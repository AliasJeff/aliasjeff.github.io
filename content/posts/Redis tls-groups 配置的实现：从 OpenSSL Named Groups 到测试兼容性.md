+++
title = "Redis tls-groups 配置的实现：从 OpenSSL Named Groups 到测试兼容性"
date = "2026-08-20"

[taxonomies]
tags=["Redis", "TLS", "OpenSSL", "问题排查"]

[extra]
comment = true
+++

PR: https://github.com/redis/redis/pull/15556

Redis Issue [#15502](https://github.com/redis/redis/issues/15502) 提到一个 TLS 配置缺口：Redis 已经支持配置 `tls-protocols`、`tls-ciphers` 和 `tls-ciphersuites`，但没有对应的选项来控制 OpenSSL TLS handshake 中使用的 group / curve preferences。对于需要满足平台 TLS 策略、合规要求，或者希望启用特定 PQC group 的部署来说，这会让 Redis 的 TLS 配置不够完整。

这次 PR 最终引入的是 `tls-groups`，不是最初 issue 里提到的 `tls-curve-preferences`。它用于设置 OpenSSL named groups 列表，并把这个配置应用到 Redis server TLS context，以及 Redis 自身作为 TLS client 时使用的 context，例如 replication、cluster 和其他 server-to-server TLS 连接。

这篇文章会从需求背景开始，解释为什么这个选项叫 `tls-groups`，然后展开服务端配置、OpenSSL API 兼容、`redis-cli` / `redis-benchmark` 支持、测试设计，以及几个容易踩到的边界问题。

## 问题背景

Redis TLS 原本已经有几类可配置项：

```text
tls-protocols      控制 TLS 协议版本，例如 TLSv1.2 / TLSv1.3
tls-ciphers        控制 TLSv1.2 及以下使用的 cipher list
tls-ciphersuites   控制 TLSv1.3 ciphersuites
```

这些选项能覆盖协议版本和 cipher 选择，但不能控制 TLS key exchange 过程中使用哪些 named groups。

在 OpenSSL 里，named groups 可以包括传统 ECC 曲线，例如：

```text
X25519
prime256v1
secp384r1
```

也可以包括新版本 OpenSSL 支持的其他 group。比如在支持 PQC hybrid group 的 OpenSSL 版本里，部署方可能希望配置：

```text
X25519MLKEM768:X25519:prime256v1
```

这里的核心诉求不是 Redis 自己理解每一个 group 的密码学含义，而是 Redis 要提供一个入口，把部署方指定的策略传给 OpenSSL。最终是否接受某个 group name、是否能和 peer 协商成功，由链接的 OpenSSL 版本和对端能力共同决定。

## Named Groups 是什么

TLS handshake 中，client 和 server 需要协商 key exchange 使用的 group。这个 group 决定了 ECDHE 或其他 key exchange 机制使用的曲线 / 参数集合。

如果双方没有共同 group，TLS handshake 会失败。可以把它简化理解成：

```text
server: prime256v1
client: secp384r1
=> 没有交集，handshake failure

server: prime256v1:X25519
client: X25519:secp384r1
=> 交集是 X25519，handshake 可以继续
```

这和 cipher / ciphersuite 是不同维度。cipher 控制加密算法组合，group 控制 key exchange 参数。一个 TLS 策略通常需要同时约束协议版本、cipher、ciphersuite 和 group。

## 为什么叫 tls-groups

最初需求里使用的是 curve preferences 这个表述，但最终配置名选择了 `tls-groups`。

这个命名更贴近现代 OpenSSL 和 TLS 语义。早期 OpenSSL API 叫：

```c
SSL_CTX_set1_curves_list()
```

后来 OpenSSL 使用更通用的命名：

```c
SSL_CTX_set1_groups_list()
```

原因也很直接：TLS named groups 不一定只表示传统 elliptic curves。继续叫 `curve-preferences` 会把语义限制在 curve 上，而 `tls-groups` 更准确，也和 OpenSSL 新 API 的命名一致。

所以这次实现里，对用户暴露的是：

```conf
tls-groups X25519:prime256v1
```

内部优先使用 `SSL_CTX_set1_groups_list()`；如果 OpenSSL 头文件里没有这个新 API，但有旧的 `SSL_CTX_set1_curves_list()`，就回退到旧 API。

## 配置层设计

Redis 的 TLS context 配置集中在 `redisTLSContextConfig` 里。这次新增了一个字段：

```c
typedef struct redisTLSContextConfig {
    char *cert_file;
    char *key_file;
    char *key_file_pass;
    char *client_cert_file;
    char *client_key_file;
    char *client_key_file_pass;
    char *dh_params_file;
    char *ca_cert_file;
    char *ca_cert_dir;
    char *protocols;
    char *ciphers;
    char *ciphersuites;
    char *groups;
    int prefer_server_ciphers;
    ...
} redisTLSContextConfig;
```

然后在配置表里加入 `tls-groups`：

```c
createStringConfig("tls-groups", NULL, MODIFIABLE_CONFIG, EMPTY_STRING_IS_NULL,
                   server.tls_ctx_config.groups, NULL, NULL, applyTlsCfg),
```

这里有几个细节：

- `MODIFIABLE_CONFIG`：允许通过 `CONFIG SET tls-groups ...` 运行时修改；
- `EMPTY_STRING_IS_NULL`：设置为空字符串时回到默认状态；
- `applyTlsCfg`：修改后重新应用 TLS context 配置；
- 没有额外 validator：group name 直接交给 OpenSSL 校验。

不在 Redis 层维护一份 group 名称白名单，是一个重要选择。OpenSSL 版本之间支持的 group 会变化，尤其是 PQC / hybrid group 这类能力。如果 Redis 自己过滤，反而可能阻止新 OpenSSL 支持的新 group 被使用。

因此 Redis 只做传递：

```text
用户配置 tls-groups
  -> Redis 保存字符串
  -> 创建 / 重建 SSL_CTX 时传给 OpenSSL
  -> OpenSSL 判断字符串是否合法
```

如果 OpenSSL 不接受这个列表，TLS context 更新失败。比如：

```text
CONFIG SET tls-groups invalid-group
=> Unable to update TLS configuration
```

这比 Redis 静默忽略配置更安全。TLS 策略类配置一旦写错，应该显式失败，而不是让用户误以为策略已经生效。

## OpenSSL API 兼容

`tls-groups` 的实现需要兼容不同 OpenSSL 版本。

新 API 是：

```c
SSL_CTX_set1_groups_list(ctx, list)
```

旧 API 是：

```c
SSL_CTX_set1_curves_list(ctx, list)
```

二者语义接近，但命名不同。Redis 的服务端 TLS 代码用一组宏统一这件事：

```c
#if defined(TLS_NO_GROUPS)
#define CONN_TLS_SUPPORTS_GROUPS 0
#elif defined(SSL_CTX_set1_groups_list)
#define CONN_TLS_SUPPORTS_GROUPS 1
#define redisTlsCtxSetGroupsList(ctx, list) SSL_CTX_set1_groups_list((ctx), (list))
#elif defined(SSL_CTX_set1_curves_list)
#define CONN_TLS_SUPPORTS_GROUPS 1
#define redisTlsCtxSetGroupsList(ctx, list) SSL_CTX_set1_curves_list((ctx), (list))
#else
#error "tls-groups requires OpenSSL with SSL_CTX_set1_groups_list or SSL_CTX_set1_curves_list. Define TLS_NO_GROUPS to build without TLS groups."
#endif
```

这里有两层策略。

第一层是默认严格：如果当前 OpenSSL 头文件既没有 `SSL_CTX_set1_groups_list`，也没有 `SSL_CTX_set1_curves_list`，Redis TLS build 会直接失败。

这看起来有点强硬，但对 TLS 策略配置是合理的。否则 Redis 编译成功了，用户也看到了 `tls-groups` 配置项，但实际配置被忽略，会产生错误的安全预期。

第二层是显式 opt-out：如果构建者确实要在不支持这个 API 的 OpenSSL 上构建 Redis，可以定义：

```bash
make BUILD_TLS=yes CFLAGS=-DTLS_NO_GROUPS
```

此时 `CONN_TLS_SUPPORTS_GROUPS` 为 0，功能被编译掉。但如果运行时设置了非空 `tls-groups`，TLS context 配置仍然会失败，而不是悄悄忽略。

这条规则可以总结成：

```text
默认：不支持 API 就编译失败
显式 TLS_NO_GROUPS：允许编译，但设置 tls-groups 会失败
```

## 应用到 TLS Context

真正把配置应用到 OpenSSL 的位置在 TLS context 创建流程里。

简化后的逻辑是：

```c
if (ctx_config->groups) {
#if CONN_TLS_SUPPORTS_GROUPS
    if (!redisTlsCtxSetGroupsList(ctx, ctx_config->groups)) {
        serverLog(LL_WARNING, "Failed to configure TLS groups: %s", ctx_config->groups);
        goto error;
    }
#else
    serverLog(LL_WARNING, "Failed to configure TLS groups: not supported by this build");
    goto error;
#endif
}
```

也就是说，只有配置非空时才调用 OpenSSL API。配置为空时，Redis 不设置 group preference，让 OpenSSL 使用默认策略。

这个配置会影响 Redis 创建的 TLS context，包括：

- 对外提供 TLS 服务的 server context；
- Redis 自身作为 TLS client 使用的 client context；
- replication、cluster 等 server-to-server TLS 连接。

这也意味着它是一个全局 TLS 策略配置。配置过窄时，可能不只是普通 client 连不上，replica、cluster node 或其他 Redis 内部 TLS 连接也可能因为没有共同 group 而 handshake 失败。

例如只允许：

```conf
tls-groups X25519MLKEM768
```

如果对端 OpenSSL 不支持这个 group，就会失败。实际部署时通常更稳妥的是提供 fallback list：

```conf
tls-groups X25519MLKEM768:X25519:prime256v1
```

Redis 不会强制用户这么写，因为这是部署策略问题，不是 Redis 能替用户判断的事情。

## redis-cli 和 redis-benchmark

只支持服务端配置还不够。测试和排查 TLS group 策略时，也需要 Redis 自带客户端能指定 group list。

因此这次同时给 `redis-cli` 和 `redis-benchmark` 增加了：

```bash
--tls-groups <list>
```

内部把它放进公共的 `cliSSLconfig`：

```c
typedef struct cliSSLconfig {
    char *sni;
    char *cacert;
    char *cacertdir;
    int skip_cert_verify;
    char *cert;
    char *key;
    char *ciphers;
    char *ciphersuites;
    char *groups;
} cliSSLconfig;
```

客户端侧也需要处理 OpenSSL API 兼容，但行为和服务端略有不同。

服务端默认采用编译失败策略，是因为 `tls-groups` 是正式配置项，不能让用户误以为配置生效。命令行工具则更接近 `--tls-ciphersuites` 的行为：只有当前构建支持时，才展示并接受这个参数。

所以 `cli_common.h` 里有独立的能力宏：

```c
#if defined(TLS_NO_GROUPS)
#define CLI_TLS_SUPPORTS_GROUPS 0
#elif defined(SSL_CTX_set1_groups_list)
#define CLI_TLS_SUPPORTS_GROUPS 1
#define cliSslCtxSetGroupsList(ctx, list) SSL_CTX_set1_groups_list((ctx), (list))
#elif defined(SSL_CTX_set1_curves_list)
#define CLI_TLS_SUPPORTS_GROUPS 1
#define cliSslCtxSetGroupsList(ctx, list) SSL_CTX_set1_curves_list((ctx), (list))
#else
#define CLI_TLS_SUPPORTS_GROUPS 0
#endif
```

`redis-cli` 和 `redis-benchmark` 的参数解析、help 文案都被包在 `CLI_TLS_SUPPORTS_GROUPS` 下：

```c
#if CLI_TLS_SUPPORTS_GROUPS
} else if (!strcmp(argv[i],"--tls-groups") && !lastarg) {
    config.sslconfig.groups = argv[++i];
}
#endif
```

这样做的结果是：

```text
支持 OpenSSL named group API 的构建
  -> redis-cli / redis-benchmark 接受 --tls-groups

不支持或显式 TLS_NO_GROUPS 的构建
  -> help 不展示 --tls-groups
  -> 参数解析不接受 --tls-groups
```

客户端公共 TLS 初始化逻辑再根据 `config.groups` 设置 SSL context：

```c
#if CLI_TLS_SUPPORTS_GROUPS
if (config.groups) {
    if (!cliSslCtxSetGroupsList(ssl_ctx, config.groups)) {
        *err = "Error while configuring TLS groups";
        goto error;
    }
}
#endif
```

这里没有保留“不支持时返回 TLS groups are not supported”的运行时分支，因为在不支持的构建里，参数本身不会被解析进 `config.groups`。运行时 fallback 分支既不可达，也会让能力隐藏策略变得不一致。

## 测试设计

这个功能的测试覆盖了几层。

第一层是配置 introspection，确认 `tls-groups` 出现在 Redis 配置列表里：

```tcl
tls-protocols
tls-ciphers
tls-ciphersuites
tls-groups
tls-port
```

第二层是配置值校验：

```tcl
assert_equal {OK} [r CONFIG SET tls-groups "prime256v1"]

catch {r CONFIG SET tls-groups "invalid-group"} e
assert_match {*Unable to update TLS configuration*} $e

r CONFIG SET tls-groups ""
```

这里 Redis 不自己解析 group name，但 OpenSSL 会在设置 SSL context 时校验。因此无效 group 应该导致 TLS configuration update 失败。

第三层是 handshake 行为测试。为了让 group negotiation 的结果稳定可观察，测试强制使用 TLSv1.2 和一个 ECDHE cipher：

```tcl
set ecdhe_ciphers ECDHE-RSA-AES128-GCM-SHA256
r CONFIG SET tls-protocols TLSv1.2
r CONFIG SET tls-ciphers $ecdhe_ciphers
r CONFIG SET tls-groups prime256v1
```

成功路径里，server 和 client 都允许 `prime256v1`：

```tcl
assert_equal {PONG} [tls_redis_cli [srv 0 host] [srv 0 port] prime256v1]
```

失败路径里，server 只允许 `prime256v1`，client 只允许 `secp384r1`：

```tcl
assert_equal 1 [catch {tls_redis_cli [srv 0 host] [srv 0 port] secp384r1} e]
assert_match {*sslv3 alert handshake failure*} $e
```

这里有几个细节值得展开。

最开始可以想到用 `openssl s_client` 做测试，但 `s_client` 的退出码和输出文本在不同 OpenSSL 版本、不同平台上并不完全稳定。用它来判断“握手成功 / 失败”容易让测试和 OpenSSL CLI 文案绑定。

最终测试改成用 Redis 自己新增的 `redis-cli --tls-groups`。这样一方面验证了 server 的 `tls-groups`，另一方面也验证了 `redis-cli` 的同名参数，测试目标更直接。

另外，失败路径不再只写一个负向断言，例如“输出里不能有 PONG”。负向断言太弱，因为很多错误都可能不包含 `PONG`。测试应该确认失败原因确实是 TLS group negotiation 导致的 handshake failure，所以匹配了具体错误：

```tcl
assert_match {*sslv3 alert handshake failure*} $e
```

成功路径也直接比较 `PONG`，没有额外 `string trim`。测试不应该隐藏输出格式变化；如果 `redis-cli` 返回的结果不是预期值，应该直接暴露出来。

第四层是 `redis-benchmark` 集成测试，确认 benchmark 客户端能带 `--tls-groups` 正常连接并完成请求：

```tcl
set cmd [redisbenchmark $master_host $master_port \
    "-r 50 -t set -n 1000 --tls-groups prime256v1"]
common_bench_setup $cmd
assert_match  {*calls=1000,*} [cmdstat set]
```

这覆盖了另一个 Redis 自带 TLS client 工具。

## 文档和 redis.conf

文档层面主要补了两处。

`redis.conf` 里增加示例：

```conf
# Configure TLS group preferences. See the SSL_CTX_set1_groups_list(3ssl)
# manpage for more information about the syntax of this string.
#
# tls-groups prime256v1
```

`TLS.md` 里说明了：

- `tls-groups` 控制 OpenSSL named groups；
- 配置值直接传给 OpenSSL；
- Redis 不过滤 group name；
- client 和 server 必须至少有一个共同 group；
- 需要 OpenSSL 1.0.2+ 的 `SSL_CTX_set1_curves_list()`，新版本也支持 `SSL_CTX_set1_groups_list()`；
- 老 OpenSSL 默认编译失败；
- 可以通过 `TLS_NO_GROUPS` 显式编译掉；
- 编译掉后，设置 `tls-groups` 会导致 TLS context 配置失败；
- `redis-cli` / `redis-benchmark` 只有在构建支持时才暴露 `--tls-groups`。

这些说明很关键。TLS 策略配置如果只改代码不写清楚边界，用户很容易误解为 Redis 支持某个固定 group 集合，或者误以为配置只影响普通客户端连接。

## 几个设计边界

### Redis 不判断具体 group 是否“安全”

Redis 只负责把字符串传给 OpenSSL。比如：

```conf
tls-groups X25519MLKEM768
```

如果当前 OpenSSL 支持这个 group，配置可以成功；如果不支持，配置失败。

Redis 不维护安全推荐列表，也不判断用户是否应该配置 fallback group。这属于部署安全策略，不应该硬编码在 Redis 里。

### 单个 group 是合法配置

只配置一个 group 没有问题：

```conf
tls-groups prime256v1
```

它的语义就是只允许这个 group。风险是对端必须也支持它，否则 handshake 失败。

这和只配置一个 cipher 类似，配置本身合法，但兼容性由部署方负责。

### 编译失败比静默忽略更合适

TLS 配置是安全策略。对于安全策略来说，最危险的行为不是报错，而是用户以为策略生效了，但实际没有生效。

因此服务端路径在没有 OpenSSL API 时默认编译失败。只有显式定义 `TLS_NO_GROUPS`，才允许构建出不支持该功能的 Redis。

### CLI 采用能力隐藏

`redis-cli` / `redis-benchmark` 是工具，不是 Redis server 配置项。它们跟随 `--tls-ciphersuites` 一类选项的模式：构建支持时展示并解析，不支持时隐藏。

这避免了用户在不支持的构建里看到一个实际不可用的命令行参数。

### 测试尽量验证 Redis 行为，而不是 OpenSSL CLI 文案

TLS handshake 失败时，OpenSSL 不同版本的输出可能有差异。测试最好验证 Redis 工具链的行为，而不是依赖 `openssl s_client` 的退出码和展示文本。

因此最终测试使用 `redis-cli --tls-groups` 发 `PING`，并在失败场景里匹配具体 handshake failure。

## 最终效果

合并后，Redis 可以这样配置 TLS named groups：

```conf
tls-groups X25519:prime256v1
```

运行时可以修改：

```text
CONFIG SET tls-groups prime256v1
CONFIG SET tls-groups ""
```

无效 group 会失败：

```text
CONFIG SET tls-groups invalid-group
=> Unable to update TLS configuration
```

`redis-cli` 可以指定 client 侧 group list：

```bash
src/redis-cli \
  --tls \
  --insecure \
  --tls-groups prime256v1 \
  PING
```

`redis-benchmark` 也可以指定：

```bash
src/redis-benchmark \
  --tls \
  --insecure \
  --tls-groups prime256v1 \
  -t set \
  -n 1000
```

如果 client 和 server 没有共同 group，handshake 会失败。这不是 Redis 的额外限制，而是 TLS group negotiation 的自然结果。

## 总结

`tls-groups` 看起来只是一个新配置项，但它涉及几类工程取舍：

- 命名上选择 `groups`，避免把语义限制在传统 curves；
- 配置值直接传给 OpenSSL，保留新 OpenSSL group 的可扩展性；
- 服务端默认编译失败，避免 TLS 策略被静默忽略；
- 通过 `TLS_NO_GROUPS` 提供显式 opt-out；
- `redis-cli` / `redis-benchmark` 采用能力隐藏，保持命令行体验一致；
- 测试使用 Redis 自带 CLI 验证真实行为，减少对 OpenSSL CLI 输出的依赖；
- failure case 匹配明确的 handshake failure，而不是只做弱负向断言。

最终这个改动补齐了 Redis TLS 配置中的一个缺口，让协议版本、cipher、ciphersuite、named groups 这几类 TLS 策略都可以在 Redis 层配置，并且能通过 Redis 自带工具进行验证。

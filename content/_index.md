+++
template = "homepage.html"

[extra]
comment = true
+++

<style>
.current-date {
    color: #7a7a7a;
}

.homepage-hero {
    text-align: center;
    padding: 2rem 0;
}

.homepage-hero-title {
    font-size: 2rem;
    margin-bottom: 2rem;
}

.homepage-hero-subtitle {
    font-size: 1.25rem;
    margin-bottom: 1rem;
}

.homepage-navigation {
    margin: 0 0 2rem 0;
    padding: 0 0 2rem 0;
}

.homepage-navigation h2 {
    text-align: center;
    margin-bottom: 2rem;
    font-size: 2rem;
}

.nav-grid {
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(300px, 1fr));
    gap: 1.5rem;
    max-width: 1200px;
    margin: 0 auto;
    padding: 0 1rem;
}

.nav-card {
    border: 1px solid var(--border-color, #e9ecef);
    border-radius: 12px;
    padding: 1.5rem;
    text-align: center;
    transition: all 0.3s ease;
    box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.nav-card:hover {
    transform: translateY(-4px);
    box-shadow: 0 8px 25px rgba(0, 0, 0, 0.15);
}

.nav-card h3 {
    margin: 0 0 0.5rem 0;
    font-size: 1.25rem;
}

.nav-card p {
    margin: 0 0 1rem 0;
    font-size: 0.9rem;
    line-height: 1.4;
}

.nav-link {
    display: inline-block;
    padding: 0.5rem 1rem;
    border-radius: 6px;
    text-decoration: none;
    font-weight: 500;
    transition: background-color 0.3s ease;
}

.nav-link:hover {
    text-decoration: none;
}

/* Dark theme adjustments */
@media (prefers-color-scheme: dark) {
    .nav-card {
        border-color: var(--border-color, #4a5568);
    }
    
    .nav-card:hover {
        border-color: var(--accent-color, #63b3ed);
    }
}

/* Mobile responsiveness */
@media (max-width: 768px) {
    .nav-grid {
        grid-template-columns: 1fr;
        gap: 1rem;
        padding: 0 0.5rem;
    }
    
    .nav-card {
        padding: 1rem;
    }
    
    .homepage-hero-title {
        font-size: 2rem;
    }
}
</style>

<marquee direction="left"><span id="current-date" class="current-date"></span></marquee>

<div class="homepage-hero">
    <h2 class="homepage-hero-title">✨ Today's Quote:</h2>
    <img src="https://quotes-github-readme.vercel.app/api?type=horizontal&theme=dark" alt="Dev Quote" />
</div>

<div class="homepage-navigation">

<h2>Explore My Content</h2>

<div class="nav-grid">
    <div class="nav-card">
    <span>📝 Posts</span>
    <p>Technical articles, tutorials, and thoughts</p>
    <a href="/posts" class="nav-link">View All Posts →</a>
</div>

<div class="nav-card">
    <span>🚀 Projects</span>
    <p>Check out my personal development projects</p>
    <a href="/projects" class="nav-link">View My Projects →</a>
</div>

<div class="nav-card">
    <span>👋 About</span>
    <p>Learn more about me and my background</p>
    <a href="/about" class="nav-link">About Me →</a>
</div>

<div class="nav-card">
    <span>🎯 Hobbies</span>
    <p>What I do when I'm not coding</p>
    <a href="/about/hobbies" class="nav-link">My Hobbies →</a>
</div>

<div class="nav-card">
    <span>📄 Résumé</span>
    <p>Professional experience and skills</p>
    <a href="/resume" class="nav-link">View Résumé →</a>
</div>

<div class="nav-card">
    <span>🏷️ Tags</span>
    <p>Browse content by topics and categories</p>
    <a href="/tags" class="nav-link">Explore Tags →</a>
</div>
</div>
</div>

<script>
    const today = new Date();
    const options = { 
        weekday: 'long', 
        year: 'numeric', 
        month: 'long', 
        day: 'numeric' 
    };
    const formattedDate = today.toLocaleDateString('en-US', options);
    document.getElementById('current-date').textContent = formattedDate;
</script>


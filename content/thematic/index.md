---
title: Thematic
type: thematic
disallow: true
comment: false
twemoji: true
lightgallery: true
date: 2024-01-07 00:23:06
---

<style>
.nav-container {
    background: #0d1117;
    padding: 2rem;
    border-radius: 10px;
    margin: 2rem auto;
    max-width: 600px;
    border: 1px solid #30363d;
    box-shadow: 0 0 20px rgba(0,180,216,0.1);
}

.terminal-header {
    font-family: 'JetBrains Mono', monospace;
    color: #e6edf3;
    text-align: center;
    margin-bottom: 2rem;
    padding: 1rem;
    background: #161b22;
    border-radius: 6px;
}

.terminal-header::before {
    content: '$ ./navigate.sh';
    color: #00b4d8;
    display: block;
    margin-bottom: 0.5rem;
    text-align: left;
}

@keyframes blink {
    0%, 100% { opacity: 1; }
    50% { opacity: 0; }
}

.cursor {
    display: inline-block;
    width: 8px;
    height: 1em;
    background: #00b4d8;
    margin-left: 4px;
    animation: blink 1s infinite;
}

.nav-links {
    display: flex;
    flex-direction: column;
    gap: 1rem;
}

.nav-link {
    background: #161b22;
    padding: 1rem 1.5rem;
    border-radius: 6px;
    color: #58a6ff;
    text-decoration: none;
    transition: all 0.3s ease;
    border: 1px solid #30363d;
    font-family: 'JetBrains Mono', monospace;
    display: flex;
    align-items: center;
    gap: 0.8rem;
}

.nav-link:hover {
    transform: translateX(10px);
    background: #1c2129;
    box-shadow: 0 0 15px rgba(0,180,216,0.1);
    border-color: #58a6ff;
}

.nav-link i {
    color: #00b4d8;
}
</style>

<div class="nav-container">
    <div class="terminal-header">
        Welcome to CloudNative Hub_<span class="cursor"></span>
    </div>
    
* [<i class="fas fa-home"></i> ->  Openstack 搭建](/docsify-book/cloudnative-openstack/)
* [<i class="fas fa-book"></i> ->  Kubernetes 源码解读](/docsify-book/cloudnative-kubernetes/)
* [<i class="fa-solid fa-laptop-code"></i> -> Fundamentals of Computer Science](/docsify-book/special_subject) 
</div>


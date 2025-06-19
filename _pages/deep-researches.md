---
layout: default
permalink: /researches/
title: researches
nav: true
nav_order: 2
description: >
  In-depth research articles and comprehensive guides on technology leadership, 
  industry trends, and strategic insights compiled through extensive research and analysis.
---

<div class="post">
  <div class="header-bar">
    <h1>Researches</h1>
    <h2>{{ page.description }}</h2>
  </div>

  {% assign research_posts = site.researches | sort: 'date' | reverse %}
  {% if research_posts.size > 0 %}
    
    <div class="research-intro">
      <p>
        Welcome to my research collection. These are comprehensive, well-researched articles compiled through my personal lens
        that dive deep into important topics in technology leadership, career development, and industry insights. 
        Each piece represents hours of research, analysis, and synthesis of information from multiple sources.
      </p>
    </div>

    <div class="research-grid">
      {% for research in research_posts %}
        {% include research_card.liquid research=research %}
      {% endfor %}
    </div>

  {% else %}
    <div class="no-research">
      <p>No deep research articles available yet. Check back soon for comprehensive guides and analysis!</p>
    </div>
  {% endif %}
</div>

<style>
.research-intro {
  background: var(--global-bg-color);
  border-left: 4px solid var(--global-theme-color);
  padding: 1.5rem;
  margin: 2rem 0;
  border-radius: 0 8px 8px 0;
}

.research-grid {
  display: grid;
  gap: 2rem;
  margin-top: 2rem;
}

.research-card {
  background: var(--global-card-bg-color, #fff);
  border: 1px solid var(--global-divider-color);
  border-radius: 12px;
  padding: 2rem;
  transition: all 0.3s ease;
  box-shadow: 0 2px 8px rgba(0,0,0,0.1);
}

.research-card:hover {
  transform: translateY(-2px);
  box-shadow: 0 8px 25px rgba(0,0,0,0.15);
  border-color: var(--global-theme-color);
}

.research-header h2 {
  margin: 0 0 0.5rem 0;
  font-size: 1.5rem;
  line-height: 1.3;
}

.research-header h2 a {
  color: var(--global-text-color);
  text-decoration: none;
  transition: color 0.3s ease;
}

.research-header h2 a:hover {
  color: var(--global-theme-color);
}

.research-meta {
  display: flex;
  gap: 1rem;
  font-size: 0.9rem;
  color: var(--global-text-color-light);
  margin-bottom: 1rem;
}

.research-description {
  font-size: 1rem;
  line-height: 1.6;
  color: var(--global-text-color);
  margin-bottom: 1rem;
}

.research-tags {
  display: flex;
  flex-wrap: wrap;
  gap: 0.5rem;
  margin-bottom: 1rem;
}

.tag {
  background: var(--global-theme-color);
  color: white;
  padding: 0.25rem 0.75rem;
  border-radius: 15px;
  font-size: 0.8rem;
  font-weight: 500;
  transition: all 0.2s ease;
}

.tag:hover {
  background: var(--global-hover-color);
  transform: scale(1.05);
}

.tag.more-tags {
  background: var(--global-text-color-light);
  font-style: italic;
}

.research-type {
  background: var(--global-bg-color);
  color: var(--global-theme-color);
  border: 1px solid var(--global-theme-color);
  padding: 0.25rem 0.5rem;
  border-radius: 12px;
  font-size: 0.75rem;
  font-weight: 600;
}

.research-meta {
  display: flex;
  flex-wrap: wrap;
  gap: 1rem;
  align-items: center;
}

.research-meta span {
  display: flex;
  align-items: center;
  gap: 0.25rem;
}

.research-footer {
  border-top: 1px solid var(--global-divider-color);
  padding-top: 1rem;
}

.read-more {
  color: var(--global-theme-color);
  text-decoration: none;
  font-weight: 600;
  transition: color 0.3s ease;
}

.read-more:hover {
  color: var(--global-hover-color);
  text-decoration: underline;
}

.no-research {
  text-align: center;
  padding: 4rem 2rem;
  color: var(--global-text-color-light);
}

@media (max-width: 768px) {
  .research-card {
    padding: 1.5rem;
  }
  
  .research-meta {
    flex-direction: column;
    gap: 0.5rem;
  }
  }
</style> 
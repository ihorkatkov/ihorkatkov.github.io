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
    
    <ul class="post-list">
      {% for research in research_posts %}
        {% assign read_time = research.content | number_of_words | divided_by: 180 | plus: 1 %}
        {% assign year = research.date | date: "%Y" %}
        {% assign tags = research.tags | join: "" %}
        {% assign categories = research.categories | join: "" %}

        <li>
          <h3>
            <a class="post-title" href="{{ research.url | relative_url }}">{{ research.title }}</a>
          </h3>
          <p>{{ research.description }}</p>
          <p class="post-meta">
            {{ research.reading_time | default: read_time }} min read &nbsp; &middot; &nbsp;
            {{ research.date | date: '%B %d, %Y' }}
            {% if research.author %}
            &nbsp; &middot; &nbsp; {{ research.author }}
            {% endif %}
          </p>
          <p class="post-tags">
            <a href="{{ year | prepend: '/researches/' | prepend: site.baseurl}}">
              <i class="fa-solid fa-calendar fa-sm"></i> {{ year }}
            </a>

            {% if tags != "" %}
              &nbsp; &middot; &nbsp;
              {% for tag in research.tags %}
                <a href="{{ tag | slugify | prepend: '/researches/tag/' | prepend: site.baseurl}}">
                  <i class="fa-solid fa-hashtag fa-sm"></i> {{ tag }}
                </a>
                {% unless forloop.last %}
                  &nbsp;
                {% endunless %}
              {% endfor %}
            {% endif %}

            {% if categories != "" %}
              &nbsp; &middot; &nbsp;
              {% for category in research.categories %}
                <a href="{{ category | slugify | prepend: '/researches/category/' | prepend: site.baseurl}}">
                  <i class="fa-solid fa-tag fa-sm"></i> {{ category }}
                </a>
                {% unless forloop.last %}
                  &nbsp;
                {% endunless %}
              {% endfor %}
            {% endif %}
          </p>
        </li>
      {% endfor %}
    </ul>

  {% else %}
    <p>No research articles available yet. Check back soon for comprehensive guides and analysis!</p>
  {% endif %}
</div>


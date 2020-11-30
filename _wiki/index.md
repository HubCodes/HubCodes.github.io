---
layout  : wikiindex
title   : wiki
toc     : true
public  : true
comment : false
regenerate: true
---

## wiki items

* [[기억력의_한계]]
* [[크로스_컴파일_환경_구축하기]]

---

## blog posts
<div>
    <ul>
{% for post in site.posts %}
    {% if post.public != false %}
        <li>
            <a class="post-link" href="{{ post.url | prepend: site.baseurl }}">
                {{ post.title }}
            </a>
        </li>
    {% endif %}
{% endfor %}
    </ul>
</div>


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
* [[Unicode]]
* [[Scala_Type_Lambda가_무엇일까]]
* [[Go_토막상식]]
* [[Optional_Chaining과_await의_관계]]

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

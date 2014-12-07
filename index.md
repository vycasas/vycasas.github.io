---
layout: pages
---

{% for post in site.posts %}
{% if forloop.index <= 10 %}
[{{ post.title }}]({{ post.url }})
----------------------------------
{: .dszsummary-title}

{% capture postdate %}{{ post.date | date: '%Y %B %d' }}{% endcapture %}
{{ postdate }}
{: .dszsummary-date}

{% if post.content contains '<!--read_more-->' %}
{{ post.content | split:'<!--read_more-->' | first }}

[Read more &#8594;]({{ post.url }})
{: .dszsummary-readmore}
{% else %}
{{ post.content }}
{% endif %}

----
{: .dszsummary-split}

{% endif %}
{% endfor %}

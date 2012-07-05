---
layout: default
title: Welcome
---

### Blog Posts

<ul>
{% for post in site.posts %}
<li><span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ post.url }}">{{ post.title }}</a></li>
{% endfor %}
</ul>

### Feedback

Feel free to leave feedback as issues on the [github page](https://github.com/farproc/farproc.github.com/issues) for this site.

### Projects

Just a short list of a few of the projects I'm working on at the moment:
* nginx queuing module for high-throughput web applications
* offline homebrew - want all the goodness of homebrew....on a aeroplane?!
* petool - a simple little app to extract PE header information from windows binaries
* git-mirror-sync - a very small collection of scripts (git-hooks) to keep multiple git mirrors in sync.


---
layout: post
title:  "Jekyll post syntax is both markdown and liquid"
date:   2019-03-09 10:00:00 +0800
categories: jekyll, liquid, markdown
---

My earlier post [Ansible list manipulation](https://wangxin.github.io/ansible,/automation/2019/02/21/ansible-list-manipulation.html) has syntax errors caused Jekyll build failure. This blocked Github from updating my site for two weeks until the syntax error was fixed today.

We write Jekyll posts using markdown syntax. However, the Jekyll posts are not just processed by markdown. They are also processed by liquid. So, the Jekyll posts syntax should comply with both **markdown** and **liquid**.

For example, originally I wrote below in my post:
{% raw %}
````
```
"{{ a_list_of_string | select('match', '^Start_of_string.*') | list }}"
"{{ a_list_of_string | select('search', 'subet_of_string') | list }}"
```
````
{% endraw %}
The {% raw %}`{{`{% endraw %} and `}}` are processed by liquid template and generated error like:
```
    Liquid Warning: Liquid syntax error (line 15): Expected end_of_string but found open_round in "{{ a_list_of_string | select('match', '^Start_of_string.*') | list }}" in D:/code/personal/wangxin.github.io/_posts/2019-02-21-ansible-list-manipulation.markdown
    Liquid Warning: Liquid syntax error (line 16): Expected end_of_string but found open_round in "{{ a_list_of_string | select('search', 'subet_of_string') | list }}" in D:/code/personal/wangxin.github.io/_posts/2019-02-21-ansible-list-manipulation.markdown
```

To avoid that, we can use the `{{ "{%" }} raw %}{{ "{%" }} endraw %}` tag, for example:
````
{{ "{%" }} raw %}
```{% raw %}
"{{ a_list_of_string | select('match', '^Start_of_string.*') | list }}"
"{{ a_list_of_string | select('search', 'subet_of_string') | list }}"{% endraw %}
```
{{ "{%" }} endraw %}
````

You know what, to make the above example show, more tricks are required. You can refer to the [source code of this post](https://github.com/wangxin/wangxin.github.io/blob/master/_posts/2019-03-09-jekyll-liquid-syntax.markdown) for more details.

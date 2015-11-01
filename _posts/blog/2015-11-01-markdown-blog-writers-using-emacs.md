---
layout: post
title: Markdown Blog Writers Using Emacs
author: geyslan
modified:
categories: blog
excerpt: ""
tags: [markdown, emacs, elisp]
share: true
image:
  feature:
  credit: #name of the person or site you want to credit
  creditlink: #url to their site or licensing
date: 2015-11-01T09:25:04-03:00
---

Writing posts using markdown is an easy task. But why don't make the
whole process easier? Emacs users would thank for the
**markdown-mode** existence.

After downloading and loading the
[markdown-mode.el](https://github.com/jrblevin/markdown-mode/blob/master/markdown-mode.el)
in Emacs you just need configure the latter to **autoload** the
markdown-mode when you open your markdown files (e.g. *.md* and
*.markdown*).

{% highlight elisp %}
(autoload 'markdown-mode "markdown-mode"
   "Major mode for editing Markdown files" t)
(add-to-list 'auto-mode-alist '("\\.markdown\\'" . markdown-mode))
(add-to-list 'auto-mode-alist '("\\.md\\'" . markdown-mode))
{% endhighlight %}

For usage see
[Emacs Markdown Mode](http://jblevins.org/projects/markdown-mode/).

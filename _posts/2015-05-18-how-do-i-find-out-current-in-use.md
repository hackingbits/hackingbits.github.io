---
layout: post
title: How to find out the current (in use) XServer DISPLAY number!
date: '2015-05-18T16:42:00.002-03:00'
author: geyslan
tags: [xorg, linux, bash, command line, english]
modified_time: '2015-05-19T21:57:31.801-03:00'
blogger_id: tag:blogger.com,1999:blog-2541885528459487831.post-3303590453568159596
blogger_orig_url: http://www.hackingbits.com/2015/05/how-do-i-find-out-current-in-use.html
---

Well, just the hack of the day.

<!--more-->

{% highlight console %}
$ ps u | awk -v tty=$(cat /sys/class/tty/tty0/active) '$0 ~ tty { print $2  }' | while read pid && [[ "$display" == "" ]]; do  display="$(awk -v RS='\0' -F= '$1=="DISPLAY" { print $2 }' /proc/$pid/environ)"; if [ "$display" != "" ]; then echo "$display"; fi; done 2>/dev/null
:0
{% endhighlight %}

Can you improve it? So tell me how. o/

The discussion about the topic on [Quora](http://www.quora.com/How-do-I-find-out-the-current-in-use-XServer-DISPLAY-number) and [Linkedin](https://www.linkedin.com/grp/post/65688-6005477964035211267).

###Edited with improvement

[http://www.commandlinefu.com/commands/view/14259/find-out-the-active-xorg-server-display-number-from-outside](http://www.commandlinefu.com/commands/view/14259/find-out-the-active-xorg-server-display-number-from-outside)

{% highlight console %}
$ for p in $(pgrep -t $(cat /sys/class/tty/tty0/active)); do d=$(awk -v RS='0' -F= '$1=="DISPLAY" { print $2 }' /proc/$p/environ 2>/dev/null); [[ -n $d ]] && break; done; echo $d
{% endhighlight %}

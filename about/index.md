---
layout: page
title: About
excerpt:
modified: 2014-08-08T19:44:38.564948-04:00
image:
  feature:
  credit:
  creditlink:
---

Hacking bits is the place where some crazy minds can outrageously express their hallucinations coding or writing about computer related topics.

Be invited to **[help](http://github.com/hackingbits)** their minds get healed.
<br /><br />

## Meet the twisted ones
{: style="text-align: center;"}
<br />

{% for author in site.data.authors %}
  <center><div itemscope itemtype="http://schema.org/Person">
  {% if author[1].avatar contains 'http' %}
    <img src="{{ author[1].avatar }}" class="bio-photo" alt="{{ author[1].name }} bio photo">
  {% else %}
    <img src="{{ site.url }}/images/{{ author[1].avatar }}" class="bio-photo" alt="{{ author[1].name }} bio photo">
  {% endif %}

  <h3 itemprop="name">{{ author[1].name }}</h3>
    <p>{{ author[1].bio }}</p>
    {% if author[1].web %}<a href="{{ author[1].web }}" class="author-social" target="_blank"><i class="fa fa-fw fa-globe"></i> Web</a>{% endif %}    
    {% if author[1].email %}<a href="mailto:{{ author[1].email }}" class="author-social" target="_blank"><i class="fa fa-fw fa-envelope-square"></i> Email</a>{% endif %}
    {% if author[1].twitter %}<a href="http://twitter.com/{{ author[1].twitter }}" class="author-social" target="_blank"><i class="fa fa-fw fa-twitter-square"></i> Twitter</a>{% endif %}
    {% if author[1].facebook %}<a href="http://facebook.com/{{ author[1].facebook }}" class="author-social" target="_blank"><i class="fa fa-fw fa-facebook-square"></i> Facebook</a>{% endif %}
    {% if author[1].google.plus %}<a href="http://plus.google.com/+{{ author[1].google.plus }}" class="author-social" target="_blank"><i class="fa fa-fw fa-google-plus-square"></i> Google+</a>{% endif %}
    {% if author[1].linkedin %}<a href="http://linkedin.com/in/{{ author[1].linkedin }}" class="author-social" target="_blank"><i class="fa fa-fw fa-linkedin-square"></i> LinkedIn</a>{% endif %}
    {% if author[1].xing %}<a href="http://www.xing.com/profile/{{ author[1].xing }}" class="author-social" target="_blank"><i class="fa fa-fw fa-xing-square"></i> XING</a>{% endif %}
    {% if author[1].instagram %}<a href="http://instagram.com/{{ author[1].instagram }}" class="author-social" target="_blank"><i class="fa fa-fw fa-instagram"></i> Instagram</a>{% endif %}
    {% if author[1].tumblr %}<a href="http://{{ author[1].tumblr }}.tumblr.com" class="author-social" target="_blank"><i class="fa fa-fw fa-tumblr-square"></i> Tumblr</a>{% endif %}
    {% if author[1].github %}<a href="http://github.com/{{ author[1].github }}" class="author-social" target="_blank"><i class="fa fa-fw fa-github"></i> Github</a>{% endif %}
    {% if author[1].stackoverflow %}<a href="http://stackoverflow.com/users/{{ author[1].stackoverflow }}" class="author-social" target="_blank"><i class="fa fa-fw fa-stack-overflow"></i> Stackoverflow</a>{% endif %}
    {% if author[1].lastfm %}<a href="http://lastfm.com/user/{{ author[1].lastfm }}" class="author-social" target="_blank"><i class="fa fa-fw fa-music"></i> Last.fm</a>{% endif %}
    {% if author[1].dribbble %}<a href="http://dribbble.com/{{ author[1].dribbble }}" class="author-social" target="_blank"><i class="fa fa-fw fa-dribbble"></i> Dribbble</a>{% endif %}
    {% if author[1].pinterest %}<a href="http://www.pinterest.com/{{ author[1].pinterest }}" class="author-social" target="_blank"><i class="fa fa-fw fa-pinterest"></i> Pinterest</a>{% endif %}
    {% if author[1].foursquare %}<a href="http://foursquare.com/{{ author[1].foursquare }}" class="author-social" target="_blank"><i class="fa fa-fw fa-foursquare"></i> Foursquare</a>{% endif %}
    {% if author[1].steam %}<a href="http://steamcommunity.com/id/{{ author[1].steam }}" class="author-social" target="_blank"><i class="fa fa-fw fa-steam-square"></i> Steam</a>{% endif %}
    {% if author[1].youtube %}<a href="https://youtube.com/user/{{ author[1].youtube }}" class="author-social" target="_blank"><i class="fa fa-fw fa-youtube-square"></i> Youtube</a>{% endif %}
    {% if author[1].soundcloud %}<a href="http://soundcloud.com/{{ author[1].soundcloud }}" class="author-social" target="_blank"><i class="fa fa-fw fa-soundcloud"></i> Soundcloud</a>{% endif %}
    {% if author[1].weibo %}<a href="http://www.weibo.com/{{ author[1].weibo }}" class="author-social" target="_blank"><i class="fa fa-fw fa-weibo"></i> Weibo</a>{% endif %}
    {% if author[1].flickr %}<a href="http://www.flickr.com/{{ author[1].flickr }}" class="author-social" target="_blank"><i class="fa fa-fw fa-flickr"></i> Flickr</a>{% endif %}
    {% if author[1].codepen %}<a href="http://codepen.io/{{ author[1].codepen }}" class="author-social" target="_blank"><i class="fa fa-fw fa-codepen"></i> CodePen</a>{% endif %}    
  </div>
  </center>  
  ---
  <br />
{% endfor %}

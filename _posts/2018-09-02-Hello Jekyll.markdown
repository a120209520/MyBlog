---
layout: post
title: Hello Jekyll
date:   2018-09-02 21:56
comments: true
description: demo
tags:
- GWT
---

记录搭建Blog时常用的配置和格式：

<br><br>
<hr>

>**001——文字**

#### header
### header
## header
# header

[超链接](https://github.com/a120209520)

[带标题的超链接](https://github.com/a120209520)

- 带个标点

普通文本中间插入一个`特殊标记`，或者超链接[github](https://github.com/a120209520)，还有 *斜体*，**粗体**

{% highlight bash %}
插入一个文本框
{% endhighlight bash %}

>插入另一种文本框

>Note
{: .note}

>Warning
{: .note .warning}

键盘图标：`Ctrl`{: .key} + `A`{: .key}

<br><br>
<hr>

>**002——代码**

>XML
{:.filename}
{% highlight xml linenos%}
{% raw %}
<?xml version="1.0" encoding="UTF-8"?>
<rss version="2.0" xmlns:atom="http://www.w3.org/2005/Atom">
  <channel>
    <title>{{ site.name }}</title>
    <description>{{ site.description }}</description>
    <link>{{site.baseurl | prepend:site.url}}</link>
    <atom:link href="{{site.baseurl | prepend:site.url}}/feed.xml" rel="self" type="application/rss+xml" />
    {% for post in site.posts limit:10 %}
      <item>
        <title>{{ post.title }}</title>
        <description>{{ post.content | xml_escape }}</description>
        <pubDate>{{ post.date | date: "%a, %d %b %Y %H:%M:%S %z" }}</pubDate>
        <link>{{post.url | prepend:site.baseurl | prepend:site.url}}</link>
        <guid isPermaLink="true">{{post.url | prepend:site.baseurl | prepend:site.url}}</guid>
      </item>
    {% endfor %}
  </channel>
</rss>
{% endraw %}
{% endhighlight xml %}

>JSON
{:.filename}
{% highlight json linenos%}
{"employees":[
    {"firstName":"John", "lastName":"Doe"},
    {"firstName":"Anna", "lastName":"Smith"},
    {"firstName":"Peter", "lastName":"Jones"}
]}
{% endhighlight %}

>SQL
{:.filename}
{% highlight SQL linenos%}
select count(*) as cm_content_nodes
from alf_node nd, alf_qname qn, alf_namespace ns
where qn.ns_id = ns.id
  and nd.type_qname_id = qn.id
  and ns.uri = 'http://www.alfresco.org/model/content/1.0'
  and qn.local_name = 'content';
{% endhighlight %}

>Java
{:.filename}
{% highlight java linenos%}
private String getToken(HttpClient client) throws UnsupportedEncodingException{
  Cookie[] cookies = client.getState().getCookies();
  for (Cookie cookie : cookies){
    if (cookie.getName().equals("Alfresco-CSRFToken")){
      return URLDecoder.decode(cookie.getValue(), "UTF-8");
    }
  }
  return null;
}
{% endhighlight %}

>.java
{:.filename}
{% highlight java linenos%}

{% endhighlight %}

<br><br>
<hr>

>**003——图片**

直接显示图片： ![Battery Widget]({{ '/images/batWid2.png' | relative_url }})

卖萌：*(°0°)*
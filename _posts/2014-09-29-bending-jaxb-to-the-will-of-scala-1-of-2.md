---
layout: post
title:  "Bending JAXB to the will of Scala (Part 1 of 2)"
categories:
- scala
tags:
- scala
- jaxb
- xml
comments: true
---

Martin Krasser has written the definitive guide to [JAXB in Scala][krasserm-jaxb]. This blog post is the first in a two part series that bottoms out solutions for the corner cases that I ran into whilst appropriating all of his hard work. 

This week I'll focus on optionals. Next week I will cover lists.

<!--more-->

## Background ##

The use-case that I am exploring the background to in this series of blog posts was modelling the [Atom Syndication Format][atom-syndication-spec] in appropriately typed Scala Case Classes that serialize to the corresponding XML as cleanly and concisely as possible. My (nearly complete) implementation is available [here][atomizer-github].

I didn't consider using scala-xml, because it has had a lot of bad press over the years. Jackson and json4s both have XML bindings, but they are not flexible enough to accurately map the complete Atom Specification. The only other well supported Scala xml library that I am aware of is [scalaxb]. I have used that before and it is very good. However, 

* It relies on XSDs and although there are Atom XSDs [out there][atom-xsd-links], none are official.
* The Case Classes generated are not as friendly to use as hand rolled could be (as evidenced by [these examples of Entries generated from different XSDs][nasty-atom-scala-from-xsd] I found).
* Under the hood it is still using scala-xml

Since none of the Scala libraries perfectly matched my requirements I decided (perhaps foolishly) to try falling back to the de-facto Java standard, [JAXB][jaxb-home]. 

## Basic Marshalling ##

Basic types (strings, nested types, un-parameterised custom types) are very simple to map once you understand how to map annotations correctly in Scala (and as Martin Krasser explains very well in [his blog post][krasserm-jaxb]). 

## The Troubles with Optionals ##

Martin Krasser's [blog post][krasserm-jaxb] gives a great example of [binding an optional string][krasserm-jaxb-string-option] to a case class but problems emerge when you actually use it.

### None Behaviour on Strings ###

I expected,

{% highlight scala %}
Person("Martin Krasser", None, 30 /* haha */)
{% endhighlight %}

to serialize to

{% highlight xml %}
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<person>
  <name>Martin Krasser</name>
  <age>30</age>
</person>
{% endhighlight %}

ie to leave out the `Option[String]` username field because it was set to `None`. However, despite running JDK7u45 (Martin thought JDK7u4 and up should contain a fix for [this bug][jaxb-415]) I still got a nasty [NPE][optional-string-error-jdk7u45]. And then, even when I upgraded to the latest version of JAXB (2.2.7) the case class would marshal to XML but left in an empty element for username desptite the XMLAdapter specifically returning null.

{% highlight xml %}
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<person>
  <name>Martin Krasser</name>
  <username/>
  <age>30</age>
</person>
{% endhighlight %}

An unneccessary empty `<username/>` element breaks my requirement for _clean and concise xml_.

### Optional Inner Value-Only Types ###

More serious problems emerged when I tried to map one very specific optional inner type. Consider the following code,

{% gist caoilte/aa7839935a29713a7a51 jaxb-in-scala-1_03.scala %}

It should marshal the following case class

{% highlight scala %}
Person("Caoilte O'Connor", None)
{% endhighlight %}

to

{% highlight xml %}
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<person>
  <name>Caoilte O'Connor</name>
</person>
{% endhighlight %}

However, for me it [NPEs][optional-case-class-error] when using jdk7u45 (with or without adding JAXB 2.2.7 as an explicit dependency). The problem is very specific and I was only able to reproduce it for types with one attribute that was `@xmlValue` annotated. 

### EclipseLink MOXy : A Solution for Optionals ###

The only solution I found for these problems was to switch JAXB implementation (as advised by Blaise Doughan in this [Stack Overflow question][so-jaxb-empty-string-to-null]). [EclipseLink MOXy][eclipse-link] fixes both of the problems described above.

Switching was trivial. I just added a dependency on `"org.eclipse.persistence" % "org.eclipse.persistence.moxy" % "2.5.2"` and put the following `jaxb.properties` file 

{% highlight properties %}
javax.xml.bind.context.factory=org.eclipse.persistence.jaxb.JAXBContextFactory
{% endhighlight %}

under the same package as the case classes that I was marshalling, except in the resources directory.

As an added bonus, MOXy also doesn't print the `standalone` attribute in the xml fragment, which was breaking my requirement for _clean and concise xml_ and which Metro (the Reference Implementation) cannot be configured to remove. My person case class would now marshal as,

{% highlight xml %}
<?xml version="1.0" encoding="UTF-8"?>
<person>
  <name>Caoilte O'Connor</name>
</person>
{% endhighlight %}

Perfectly concise!

### Mapping Other People's Types to Optionals ###

The previous two examples describe the problems I found with using `OptionAdapter` to map a type that JAXB knows how to marshal/unmarshal. Taking a custom type that JAXB doesn't know how to manage (ie one you have already created your own `XmlAdapter` for) and making it optional adds a whole new dimension to the problem space. You cannot simply pass your custom type as a type parameter to the `OptionAdapter` and let JAXB figure out the chain of transformations required (although I did try). I came up with the following solution,

{% gist caoilte/aa7839935a29713a7a51 jaxb-in-scala-1_05.scala %}

There's nothing new, or even particularly pleasing about this code, but it is interesting because it was the most concise way that I was able to take an existing `XmlAdapter` (`DateTimeAdapter`) and make it optional.

I have tested that the _bug_ described in the JavaDoc of `CustomOptionAdapter` has been fixed in the trunk of EclipseLink MOXy.

## Afterthoughts ##

I decided to write about JAXB in Scala because I wanted to highlight the problems that emerge when you scratch the surface with a real life use-case. That doesn't mean I endorse the use of JAXB in Scala. I'm still very ambivalent about the library. Exactly why might become clearer in part two of this two part series when I scratch a little deeper on the problems for JAXB in Scala and talk about lists.





[krasserm-jaxb]:                 http://krasserm.blogspot.co.uk/2012/02/using-jaxb-for-xml-and-json-apis-in.html
[atom-syndication-spec]:         http://atomenabled.org/developers/syndication/
[atom-syndication-spec-person]:  http://atomenabled.org/developers/syndication/#person
[atomizer-github]:               https://github.com/caoilte/atomizer
[scalaxb]:                       http://scalaxb.org/
[atom-xsd-links]:                https://www.google.co.uk/search?q=atom+xsd&oq=atom+xsd
[nasty-atom-scala-from-xsd]:     https://gist.github.com/caoilte/aa7839935a29713a7a51#file-jaxb-in-scala-1_01-scala
[jaxb-home]:                     https://jaxb.java.net/
[jaxb-415]:                      https://java.net/jira/browse/JAXB-415
[krasserm-jaxb-string-option]:   https://gist.github.com/krasserm/1891525#file-jaxb-02-scala
[optional-string-error-jdk7u45]: https://gist.github.com/caoilte/aa7839935a29713a7a51#file-jaxb-in-scala-1_02-txt
[optional-case-class-error]:     https://gist.github.com/caoilte/aa7839935a29713a7a51#file-jaxb-in-scala-1_04-txt
[eclipse-link]:                  http://www.eclipse.org/eclipselink/moxy.php
[so-jaxb-empty-string-to-null]:  http://stackoverflow.com/questions/11894193/jaxb-marshal-empty-string-to-null-globally/11931768#11931768

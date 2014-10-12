---
layout: post
title:  "Bending JAXB to the will of Scala (Part 2 of 2)"
categories:
- scala
tags:
- scala
- jaxb
- xml
comments: true
---

This blog post is the second in a two part series that bottoms out solutions for the corner cases that I ran into whilst getting JAXB to work in Scala. I am indebted to Martin Krasser's [blog post on JAXB][krasserm-jaxb] for providing an essential first primer on the topic. 

This week I enumerate the difficulties with lists and the work arounds I found for making them manageable. You can still read the first part of the series (on optionals) [here][part-1].

<!--more-->

## Lists of the reasons to consider giving up on JAXB ##

Lists in JAXB are where the gaps between Java and Scala start to become painful. They're also where I gave up for a week in disgust and found solace in signing up for [Erik Meijer's new Haskell MOOC][edx-fp101x]. I eventually succeeded with two different ways of modeling lists, only the second of which turned out suitable for implementing the Atom specification.

### Containerized Lists ###

Martin Krasser's JAXB blog post doesn't explain how to model lists, but does link to extensive and very helpful examples in his [event sourcing project][krasserm-jaxb-working-code].

With a little bit of tweaking I managed to get them working for me,

{% gist caoilte/d0f1b86cebf68d88a04a jaxb-in-scala-2_01.scala %}

This code will turn,

{% highlight scala %}
Entry("101", 
    List(
        Person("Caoilte O'Connor"),
        Person("Martin Krasser")
    )
)
{% endhighlight %}

into,

{% highlight xml %}
<?xml version="1.0" encoding="UTF-8"?>
<entry>
    <id>101</id>
    <authors>
        <person>
            <name>Caoilte O'Connor</name>
        </person>
        <person>
            <name>Martin Krasser</name>
        </person>
    </authors>
</entry>
{% endhighlight %}

The most notable property of the xml is the nesting of list elements within an `authors` element. A lot goes on in order to achieve this containerization, so let's break it down into a list of bullets,

* JAXB doesn't know how to handle Scala Lists, but it can map Java Lists and so the `AbstractListAdapter` turns a Scala List into a container of a Java List and vice versa.
* Unfortunately one `XmlAdapter` cannot encapsulate the entire abstract transformational logic as it has for previous examples. The `AbstractList` trait describes the properties that the container requires.
* The Person companion object then implements all of the abstract types described previously. This code is entirely boiler plate and seems like an excellent opportunity to re-watch Dave Gurnell's [Macros for the Rest of Us][macros-for-rest-of-us-talk] talk.
* Finally, I have included the Person Case Class and an Entry Case class that incorporates a list of Persons. 
* The only change that I made to Martin Krasser's code was because of the MOXy bug mentioned in the previous section, which required me to reverse the order of the type parameters on `AbstractListAdapter`.

Unfortunately, despite how clever it is, this code isn't suitable for the Atom Feed use case because that requires XML to be produced without the container element and this requires a completely different approach which I will discuss in the next section.

### Mapping Scala Collections to Java Collections for Containerless Lists ###

Unfortunately the `XmlAdapter` can only be used to model the transformation for elements _in_ a list. You cannot use it to model the transformation of the list itself. We got around this in the previous section by transforming a Scala List (which JAXB doesn't recognise at all) into a _container_ of lists. If you try and turn the Scala List into a Java List, [JAXB falls over hard][list-to-list-error]. (It recognises that the property will end up as a Java List but ignores the adapter and attempts to cast the Scala List to a Java Collection directly.)

Because of this limitation the entire approach needs to be re-thought. I tried utilizing other JAXB annotations (eg [@XmlElementWrapper][jaxb-xmlelementwrapper], [@XmlPath][moxy-xmlpath] and [@XmlTransformation][moxy-xmltransformation]) but couldn't get any of them to work with Scala. I had a similar lack of success with other Scala Collection types (`Seq`, `ArraySeq` and `WrappedArray`). JAXB managed to marshall some to XML but was unable to unmarshall any back to types.

The only type which worked was the Scala `Array`. This is probably because `Array` has some unique (and to be honest, rather awkward) properties. Unlike most (perhaps all?) other Scala types, `Array` compiles down to a Java Array in byte code and is indistinguishable to Java code. This has pros and cons,

On the plus side,

* JAXB knows how to map a Java Array to xml and back again 
* Scala makes all of its Collection mapping goodies available to an `Array`
* Arrays are more performant for very large data sets (I'm clutching at straws here)

On the minus side,

* Arrays are mutable. Case Classes promote the use of immutable data because it is easier to reason with and it is a shame to be forced to give that up.
* Because Arrays are really from Java, Scala cannot override the `equals`, `hashCode` and `toString` methods. This means they are pretty much the only type in Scala to compare with _reference_ equality rather than _value_ equality. Paul Phillips gives a great explanation [here][pphillips-array-explanation] about why that's a good thing, but it still hurts and causes all sorts of problems using them in Case Classes (as we shall see). 

The comparison with reference equality can be worked around. There's a great exploration of the possibilities [here][scala-array-comparison]. Making Arrays work in a Case Class context is trickier, because a Case Class is really just [a boilerplate generator][zaugg-case-class-opinion] and the boilerplate generation makes no special exception for Arrays and the Java `equals`, `hashCode` and `toString` methods they expose. 


I spent some time figuring out how to work around this. At first I got very excited to discover that Case Classes support two parameter lists and that 
[only parameters in the first parameter section are considered for equality and hashing.][zaugg-case-class-config] (HT: Jason Zaugg, again.) However, there is no way to partially extend equals/toString/hashCode and so this didn't gain me anything as I still had to override everything. A better solution presented itself after I saw this [this manual reimplementation of a case class][manual-case-class] on a Stack Overflow question about [overriding the equals method on a case class][scalaruntime-answer]. My eventual (and I believe, optimal) solution borrowed several tricks from this discussion,

{% gist caoilte/d0f1b86cebf68d88a04a jaxb-in-scala-2_03.scala %}

This code will turn,

{% highlight scala %}
Entry("101", 
    Array(
        Person("Caoilte O'Connor"),
        Person("Martin Krasser")
    )
)
{% endhighlight %}

into,

{% highlight xml %}
<?xml version="1.0" encoding="UTF-8"?>
<entry>
    <id>101</id>
    <author>
        <name>Caoilte O'Connor</name>
    </author>
    <author>
        <name>Martin Krasser</name>
    </author>
</entry>
{% endhighlight %}

Let's consider what is going on,

* One of the lesser known features of a Case Class is that the Scala compiler  auto-generates it an implementation of the [Product Trait][product-trait-api]. The Product Trait provides an interface for programmatically accessing the attribute values of a class. The Scala Compiler then auto-generates `toString` and `hashCode` methods that rely on the Product Trait. This gives us a code seam where we can interpose our own implementation of the Product trait for a Case Class, whilst still benefitting from other auto-generated goodness.
* The only difference between our implementation of Product and the auto-generated implementation is that we override Arrays returned by the `productElement` method in a [Wrapped Array][wrapped-array-api]. The Wrapped Array is identical to a normal Array except that it overrides `toString`, `hashCode` and `equals` with more useful implementations. 
* The [manual reimplementation of a case class][manual-case-class] showed `equals` being implemented by the `ScalaRuntime._equals` method, but this is not actually the case with real case classes auto-generated by the Scala Compiler. I asked [why][my-case-class-equals-question] on the scala-user list and was relieved to learn that it was a performance optimisation. Consequently, there is nothing to stop us from overriding equals with this ourselves.

It is worth re-considering the pros and cons at this point. On the plus side,

* We've succeeded in defining a Collection containing Case Class that JAXB can marshall to XML and back again into the same Case Class.
* The Case Class we have defined doesn't break any of the conventions that the Case Class provides. It still has the same constructor, pattern-matching, equality and, product attribute properties expected.
* The hand rolled boiler plate is easy to understand and relatively simple to spot errors in.
* With a little work it might be possible to define a macro to do all of the auto-generation for us.

On the minus side,

* With larger Case Classes the boiler plate [becomes overwhelming][arrays-in-large-case-class].
* Arrays are mutable and many people are uncomfortable using mutable classes in a Case Class.
* Our `equals` method uses the `ScalaRuntime._equals` helper method which is described on the Javadoc as being _outside the API and subject to change or removal without notice_. (In practice the method is three lines long and simple to re-implement.)

## Afterthoughts ##

I have succeeded in bending JAXB to my use-case, but at what cost? If I use  JAXB Annotated Case Classes for my domain model then I,

* have to suffer the heavy miasma of annotation pollution 
* am constrained to the use of `Arrays` to describe lists and have to re-implement a large part of each case class as a result
* am at the mercy of any future Scala/Java interop bugs I find in the JAXB space (consider the map)

On reflection, JAXB is not a good fit for Scala in most scenarios - especially ones where the specification is complex or could evolve. As I have nearly completely implemented the Atom spec it isn't so bad, but I would think twice before using it again.


[part-1]:                        http://caoilte.org/scala/2014/09/29/bending-jaxb-to-the-will-of-scala-1-of-2/
[krasserm-jaxb]:                 http://krasserm.blogspot.co.uk/2012/02/using-jaxb-for-xml-and-json-apis-in.html
[edx-fp101x]:                    https://www.edx.org/course/delftx/delftx-fp101x-introduction-functional-2126
[krasserm-jaxb-working-code]:    https://github.com/krasserm/eventsourcing-example/tree/play-blog/modules/service/src/main/scala/dev/example/eventsourcing
[macros-for-rest-of-us-talk]:    https://www.parleys.com/play/53a7d2c4e4b0543940d9e542/chapter0/about
[list-to-list-error]:            https://gist.github.com/caoilte/d0f1b86cebf68d88a04a#file-jaxb-in-scala-2_02-txt
[jaxb-xmlelementwrapper]:        http://stackoverflow.com/questions/16202583/xmlelementwrapper-for-unwrapped-collections
[moxy-xmlpath]:                  http://stackoverflow.com/questions/11153599/unmarshalling-nested-collection-via-xmlpath-moxy-annotation
[moxy-xmltransformation]:        http://blog.bdoughan.com/2010/08/xmltransformation-going-beyond.html
[pphillips-array-explanation]:   http://stackoverflow.com/a/3738850/58005
[scala-array-comparison]:        http://blog.bruchez.name/2013/05/scala-array-comparison-without-phd.html
[zaugg-case-class-opinion]:      https://groups.google.com/d/msg/scala-user/buGv-DNNvWs/QzdeSlHeCIEJ
[zaugg-case-class-config]:       http://stackoverflow.com/questions/10373715/scala-ignore-case-class-field-for-equals-hascode/10373946#10373946
[scalaruntime-answer]:           http://stackoverflow.com/a/11396362/58005
[manual-case-class]: https://gist.github.com/okapies/2814608
[product-trait-api]:             http://www.scala-lang.org/api/current/index.html#scala.Product
[wrapped-array-api]:             http://www.scala-lang.org/api/current/index.html#scala.collection.mutable.WrappedArray
[my-case-class-equals-question]: https://groups.google.com/forum/#!topic/scala-user/EfEb8MPa5BM
[arrays-in-large-case-class]:    https://github.com/caoilte/atomizer/blob/755f3c6bc980a8477b2ee0806153579680661fcd/atom-model/src/main/scala/org/caoilte/atomizer/model/Atom.scala#L162

---
layout: post
title: Procs and brackets
categories: console
---

Looking things up in a hash is concise, readable and fun. :smile:

{% highlight ruby %}
dictionary['ruby']
# => 'A precious stone'
{% endhighlight %}

Finding items in an array is not that much fun. :disappointed:

{% highlight ruby %}
term = 'ruby'
(list_of_definitions.find { |definition| definition[:term] == term } || {})[:definition]
# => 'A precious stone'
{% endhighlight %}

We can have our delicious `Hash` lookup notation, for any type of lookup we want with a little `proc` trickery. :sparkles:

{% highlight ruby %}
dictionary = -> (word) {
  (
    list_of_definitions.find { |definition| definition[:term] == word } || {}
  )[:definition]
}
dictionary['ruby']
# => 'A precious stone'
{% endhighlight %}

Ruby's `proc`s support the `[]` notation for their invocation ([prc[]](http://apidock.com/ruby/v1_9_3_392/Proc/%5B%5D)). Using that knowledge we can present any complex lookup with our syntactic sugar. :+1:

---
layout: post
title: Instantiating behaviour
categories: console
---

In [last weeks post](/2015/07/01/three_little_hacks) we had a quick look at a little hack called digg.

{% highlight ruby %}
some_complex_json_rsponse.digg('preview', 'thumbnail', 0, 'id')
{% endhighlight %}

In order for this to work we need to extend each class we want to digg with the ```digg``` method.
This leads to modifying core classes; and we should [frown upon that](http://devblog.avdi.org/2008/02/23/why-monkeypatching-is-destroying-ruby/). 😠

So what else can we do?

{% highlight ruby %}
module Digg
  def self.digg(soil, path)
    # …
  end
end

Digg.digg(some_complex_json_rsponse, ['preview', 'thumbnail', 0, 'id'])
{% endhighlight %}

That would work. I bet this will very quickly lead to a “Util” class that become a bag of tricks. 👜✨
We don't want that.

So how about adding some instantiation?

{% highlight ruby %}
class Digg
  def initialize(soil)
    @soil = soil
  end

  def digg(path)
    # …
  end
end

digger = Digg.new(some_complex_json_rsponse)
digger.digg(['preview', 'thumbnail', 0, 'id'])
{% endhighlight %}

Alright! Now we can digg for multiple paths on the same set of data. 😎

But… This doesn't feel right either. 😧 The act of digging is defined by what steps are taken. (e.g.: Pick up spade, stab in ground, lift, dump, repeat). It is not defined by the soil that we digg in.

{% highlight ruby %}
array_of_nested_objects.map { |json|
  Digger.new(json).digg(['preview', 'thumbnail', 0, 'id'])
  Digger.new(json).digg(['preview', 'name'])
}
{% endhighlight %}

See! That just doesn't look good.

What if we turn that around?

{% highlight ruby %}
class Digg
  def initialize(path)
    @path = path
  end

  def digg(soil)
    # …
  end
end

digger = Digg.new(['preview', 'thumbnail', 0, 'id'])
digger.digg(some_complex_json_rsponse)
{% endhighlight %}

Now we define “how to digg” when initializing. See how much better this looks for loops:

{% highlight ruby %}
thumbnail_digger = Digg.new(['preview', 'thumbnail', 0, 'id'])
name_digger = Digger.new(['preview', 'name'])
array_of_nested_objects.map { |json|
  thumbnail_digger.digg(json)
  name_digger.digg(json)
}
{% endhighlight %}

Finally! 😌

This way of thinking about your objects, by defining behaviour on initializing, has a lot of benefits.

Not only is it faster (in this case) ([gist](https://gist.github.com/Bertg/f23a8d4467cda4173ec7)), but also more intention revealing. 📖

By instantiating behaviour you can start to compose your objects with more ease, or start thinking about caching or optimising the behaviour.

---

Let me know what you think. And if you apply this in your code I'd be happy to have a look at it.

🎩

---
layout: post
title: Three Little Hacks
categories: console
---

As I've missed my last 2 posts, I'm going to now regale you with not 1, not 2, but 3 (three!) of my favorite little hacks ruby hacks.

# Digg

Do you have a deeply nested structures you want 1 value from deep inside? Don't care if the path is broken?
Use ```digg```! I especially love this to get deeply nested data out of a JSON response.

{% highlight ruby %}

module Digg
  def digg(*path)
    path.inject(self) do |memo, key|
      (memo.respond_to?(:[]) && (memo[key] || {})) || break
    end
  end
end

Array.send :include, Digg
Hash.send :include, Digg

# Use as
some_complex_json_rsponse.digg('preview', 'thumbnail', 0, 'id')

{% endhighlight %}

<sub>(Would you believe there is no digging related emoji? Have a 📯 instead.)</sub>

# Symbol respond_to

Sometimes we switch on classes, you've seen this before right?

{% highlight ruby %}
case a_thing
  #...
when Array, Hash
  #...
when String
  #...
end
{% endhighlight %}

Not only is that ugly, but we are basing our behavior on classes, instead of behavior. Wouldn't it be cool
if we could test for methods? Well we can!

{% highlight ruby %}
case a_thing
when ->(x) { x.respond_to? :fetch }
  #...
when ->(x) { x.respond_to? :split }
  #...
end
{% endhighlight %}

Ugh... That looks horrible  😷. Let's improve that with:

{% highlight ruby %}
module SymbolRespondTo
  def ~@
    -> (o) { o.respond_to?(self) }
  end
end

Symbol.send(:include, SymbolRespondTo)
{% endhighlight %}

Adding a simple [unary method](http://www.rubyinside.com/rubys-unary-operators-and-how-to-redefine-their-functionality-5610.html) on symbol gives us a great looking switch. ✨

{% highlight ruby %}
case a_thing
when ~:fetch
  #...
when ~:split
  #...
end
{% endhighlight %}

If I could love code, this would be the code I love. 💞💗💖

# Better to_s

Often times it makes sense for the ```to_s``` value of an object to be the name or the title of the person or thing.
So out of shear laziness 😫 I wrote this little hack.

{% highlight ruby %}
module DefaultToS
  def to_s
    case self
    when ~:name then name
    when ~:title then title
    else super
    end
  end
end

ActiveRecord::Base.prepend(:to_s)
{% endhighlight %}

It might be a hack, but I like it.

---

Do you have your favorite hacks? What do you think of mine? Let me know below!

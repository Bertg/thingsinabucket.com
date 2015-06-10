---
layout: post
title: Defaults for Hash
categories: console
---

As a follow up to my [first post](/posts/2015-05-20-pocs_and_brackets) I'd like to quickly run over the "default value" related features of Ruby's ```Hash``` and explain when to use them.

# [Hash#default](http://apidock.com/ruby/v1_9_3_392/Hash/default)

{% highlight ruby %}
totals = Hash.new(0)
totals[:jeff] # => 0
totals[:ann] += 1 # => 1
{% endhighlight %}

Use this when:

* the default value is the same for every key

Do not use this when:

* you are planning on mutating the value returned when the key is missing

{% highlight ruby %}
scored = Hash.new([])
scored[:jeff] # => []
scored[:ann] << 'A+'
scored[:jeff] # => ['A+']
scored # => {}
{% endhighlight %}

💩

# [Hash#default_proc](http://apidock.com/ruby/v1_9_3_392/Hash/default_proc)

{% highlight ruby %}
fizz_bang = Hash.new { |hash, key| hash[key] = fizz_and_or_bang(key) }
fizz_bang[3] # => 'Fizz'
fizz_bang[5] # => 'Bang'
{% endhighlight %}

Use this when:

* Each key might need a different value
* And the value can be derived by the same method or proc

Don't user this when:

* You have a set of default values ( just merge your hashes )
* You are not saving the value in the hash 💾 ( you should reconsider if you need a Hash at all )

## Changing default methods

It's possible to change the default_proc and default value. The last assignment will be what's used.

{% highlight ruby %}
h = Hash.new
h[:jeff] # => nil
h.default = 'default'
h[:jeff] # => 'default'
h.default_proc = -> (*) { 'default_proc' }
h[:jeff] # => 'default_proc'
h.default = nil
h[:jeff] # => nil
h # => {}
{% endhighlight %}

# [Hash#fetch](http://apidock.com/ruby/v1_9_3_392/Hash/fetch)

New Ruby programmers probably do something like this:

{% highlight ruby %}
hash = {}
hash[:key] ||= 1
{% endhighlight %}

This works fine until some [falsy](https://gist.github.com/jfarmer/2647362) values are introduced:

{% highlight ruby %}
hash = {1 => nil, 2 => false}
hash[1] ||= 1
hash[2] ||= 2
hash # {1 => 1, 2 => 2}
{% endhighlight %}

😩

When using ```||``` your falsy values will be overwritten. This is most probably not your intention. Instead of that use ```Hash#fetch```.

{% highlight ruby %}
hash = {1 => nil, 2 => false}
hash.fetch(1) { 1 }
hash.fetch(2) { 2 }
hash # {1 => nil, 2 => false}
{% endhighlight %}

```Hash#fetch``` is best used when:

* The default value for individual keys are calculated in a very different way
* Values can be falsy

It's also great to ensure a key is present

{% highlight ruby %}
Hash.new.fetch(:a_missing_key)
# KeyError: key not found: :a_missing_key
{% endhighlight %}

Don't use it when:

* You'd like the default values to be saved in the hash
* You have a set of default values ( just merge your default into the hash )

```Hash#fetch``` also has a second notation for the default value:

{% highlight ruby %}
Hash.new.fetch(:a_missing_key, 'default value') # => 'default value'
{% endhighlight %}

I only use the notation with a block. There are no real benefits to the other notation except maybe some [performance difference in super simple cases](https://gist.github.com/Bertg/22e459aee78ca0dfbc39). In most cases though, the block version will be the better choice. It will postpone the evaluation of the default value until it's needed; and - at lest to me - it looks better. 😍

# Some idiosyncratic fun

So, what do you think this will do?

{% highlight ruby %}
Hash.new('default value') { |hash, key| hash[key] = key.to_s }
{% endhighlight %}

How about this?

{% highlight ruby %}
Hash.new.fetch('default value') { 'value' }
{% endhighlight %}

👉 Try it yourself 👈, I found it very surprising 😃

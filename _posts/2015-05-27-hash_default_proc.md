---
layout: post
title: Hash and default_proc
categories: console
---

One of my favourite features in Ruby is [Hash#default_proc](http://apidock.com/ruby/v1_9_3_392/Hash/default_proc). Let's see what it can do.

Ever seen code like this?

{% highlight ruby %}
result = {}
some_collection.map do |some_entry|
  result[some_entry] = some_expensive_operation(some_entry)
end
result
{% endhighlight %}

Stop writing that ✋.

Unless all the values in the collection will be used, the expensive calculations should be delayed until it's needed.

{% highlight ruby %}
result = Hash.new do |hash, key|
  hash[key] = some_expensive_operation(key)
end
{% endhighlight %}

That looks a lot better already! 🎉

Ruby will evaluate the code in the block when a key is missing. This makes it great tool for these types of lookups, or as a quick in memory cache.

In [last weeks post](/2015/05/20/pocs_and_brackets) we saw that we can call procs using brackets. When those operations become expensive, we can just wrap it with a Hash without changing our client code.

{% highlight ruby %}
expensive_proc = ->(v) do
  puts "Expensive calculation"
  sleep 1
  [v] * 2
end

expensive_proc_with_cache = Hash.new do |hash, key|
  hash[key] = expensive_proc[key]
end

def repeat_10_times(proc)
  10.times do |i|
    proc[ i % 2 ]
  end
end

repeat_10_times(expensive_proc) # Takes about 10 seconds
repeat_10_times(expensive_proc_with_cache) # Takes about 2 seconds
{% endhighlight %}

Great improvement! 👍

How about when you make an API call, and it returns the next and previous values as well, it would be a waste not to use them.

{% highlight ruby %}
paginated_lookup = ->(index) do
  v = "index[#{index}]"
  {
    index => v,
    (index + 1) => v + '.next',
    (index - 1) => v + '.prev',
  }
end

paginated_lookup_cache = Hash.new do |hash, key|
  hash.merge! paginated_lookup[key]
  hash[key]
end

paginated_lookup_cache[10] => "index[10]"
paginated_lookup_cache[11] => "index[10].next"
{% endhighlight %}

We can populate the Hash based on the previous calls. Sweet! 🎂

## Caveat! 🐹

The Hash#default_proc will not be called when using [Hash#fetch](http://apidock.com/ruby/v1_9_3_392/Hash/fetch). This surprised me. 👻

{% highlight ruby %}
times2 = Hash.new {|_,v| v*2 } # => {}
times2.fetch(1) # KeyError: key not found: 1
{% endhighlight %}

Please don't use this "workaround" either.

{% highlight ruby %}
times2.fetch(1) { |k| times2[k] }
{% endhighlight %}

That's just 💩.

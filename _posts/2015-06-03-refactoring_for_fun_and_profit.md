---
layout: post
title: Refactoring for fun and profit
categories: console
---

Today I will guide you through a class refactoring done in one of the projects I work on. Ideally we should be doing this test driven, but - as this is for fun - I won't today.

{% highlight ruby %}
class VirusScanner

  attr_reader :file_path

  def initialize(file_path)
    @file_path = file_path
  end

  def infected?
    !Settings['virusscan_off'] && scan != 'OK'
  end

private

  def scan
    return @scan if @scan

    command = Cocaine::CommandLine.new('clamscan', "--no-summary :file").command(file: file_path)
    _, stdout, stderr = ::Open3.popen3(command)

    @scan = parse_clamscan_output(stdout.gets, stderr.gets)
  end

  def parse_clamscan_output(stdout, stderr)
    stderr = stderr.to_s.split("\n").reject{|line| line =~ /\ALibClamAV Warning:/ }.join("\n")
    raise RuntimeError, "The 'clamscan' command line was unable to execute. #{stderr}" if !stderr.blank?

    stdout.to_s.split("\n").last.split(":")[1].strip
  end

end
{% endhighlight %}

This is a simple interface around the [clamscan](https://www.clamav.net) scanner. This code has several smells. 😷

* Turning off the virus scanner is done using a global setting
* Several external dependancies make it more difficult to port
* [Train wrecks](http://en.wikipedia.org/wiki/Method_chaining) all over the place 🚂🚋
* So many Bangs 💥

Our primary goal is to keep backward compatibility with the rest of the code base, while cleaning up the code.

# \#1 Separate the generic VirusScanner from the specific clamscan.

By doing this separation we can focus on the the VirusScanner without needing to worry about clamscan.

{% highlight ruby %}
class VirusScanner

  attr_reader :file_path

  def initialize(file_path)
    @file_path = file_path
  end

  def infected?
    return false if Settings['virusscan_off']
    ClamScan.new(scanner).infected?
  end

end

class ClamScan

  attr_reader :file_path

  def initialize(file_path)
    @file_path = file_path
  end

  def infected?
    scan != 'OK'
  end

private

  def scan
    #...
  end

  def parse_clamscan_output(stdout, stderr)
    #...
  end

end
{% endhighlight %}

Cool. 😎

# \#2 Removing the "Settings" for the virus scanner

I want to get rid of the app specific 'virusscan_off' setting. Honestly I'm ashamed I ever wrote that 😳

The first step is to make the behavior of the class configurable.

{% highlight ruby %}
class VirusScanner

  attr_reader :file_path

  def initialize(file_path, options = {})
    @file_path = file_path

    @scanner = options.fetch(:scanner) {
      ->(file_path) { ClamScan.new(scanner).infected? }
    }
  end

  def infected?
    return false if Settings['virusscan_off']

    @scanner.call(file_path)
  end

end
{% endhighlight %}

We created an easy way to understand the interface for VirusScanner's scanner. It's now also very easy to extract the 'virusscan_off' setting.

{% highlight ruby %}
scanner_with_off_switch = ->(*args) {
  Settings['virusscan_off'] ? false : ClamScan.new(scanner).infected?
}
{% endhighlight %}

This works! However! This breaks one of our earlier requirements: "keep backward compatibility". This solution also requires the developer to know what the default scanner is, and that a new smell; a smell I don't like. 😡

Let's do better!

{% highlight ruby %}
class VirusScanner

  def self.default_scanner
    @default_scanner ||= -> (file_path) { ClamScan.new(scanner).infected? }
  end

  def self.default_scanner=(scanner)
    @default_scanner = scanner
  end

  attr_reader :file_path

  def initialize(file_path, options = {})
    @file_path = file_path
    @scanner = options.fetch(:scanner) { self.class.default_scanner }
  end

  def infected?
    @scanner.call(file_path)
  end

end

old_scanner = VirusScanner.default_scanner
VirusScanner.default_scanner = ->(*args) {
  Settings['virusscan_off'] ? false : old_scanner.call(*args)
}
{% endhighlight %}

Ok. I'm pretty happy with that. 😌

Now we turn to the ```ClamScan``` class.

# \#3 Remove external dependancies

[Cocaine](https://rubygems.org/gems/cocaine) has a lot of powerful features, has tones of great use cases~~, and I had a phase where I was totally addicted to it~~. 😇

Here Cocaine is only used to safely use a file name in a shell command. 🐌 [Shellwords](http://apidock.com/ruby/Shellwords?q=Shellwords) - a standard lib - can deal with this easily.

{% highlight ruby %}
class ClamScan
#...
private

  def scan
    return @scan if @scan

    command = Shellwords.join(['clamscan', '--no-summary', file_path])
    _, stdout, stderr = ::Open3.popen3(command)

    @scan = parse_clamscan_output(stdout.gets, stderr.gets)
  end

  def parse_clamscan_output(stdout, stderr)
    stderr = stderr.to_s.split("\n").reject{|line| line =~ /\ALibClamAV Warning:/ }.join("\n")
    raise RuntimeError, "The 'clamscan' command line was unable to execute. #{stderr}" if !stderr.blank?

    stdout.to_s.split("\n").last.split(":")[1].strip
  end

end
{% endhighlight %}

# \#4 Use the correct version of Open3

[Open3.popen3](http://apidock.com/ruby/Open3/popen3) opens a stdin pipe, we don't need that at all. A better choice would be [Open3.capture3](http://apidock.com/ruby/Open3/capture3).

{% highlight ruby %}
class ClamScan
#...
private

  def scan
    return @scan if @scan

    command = Shellwords.join(['clamscan', '--no-summary', file_path])
    stdout, stderr, _ = Open3.capture3(command)

    @scan = parse_clamscan_output(stdout, stderr)
  end

  def parse_clamscan_output(stdout, stderr)
    stderr = stderr.split("\n").reject{|line| line =~ /\ALibClamAV Warning:/ }.join("\n")
    raise RuntimeError, "The 'clamscan' command line was unable to execute. #{stderr}" if !stderr.blank?

    stdout.split("\n").last.split(":")[1].strip
  end

end
{% endhighlight %}

Let us also clean the code... 🚿

{% highlight ruby %}
class ClamScan

  attr_reader :file_path

  def initialize(file_path)
    @file_path = file_path
  end

  def infected?
    @infected ||= begin
      stdout, stderr, status = Open3.capture3(command)
      !clamscan_result_safe?(stdout, stderr, status)
    end
  end

private

  def command
    Shellwords.join(['clamscan', '--no-summary', file_path])
  end

  def clamscan_result_safe?(stdout, stderr, status)
    raise_on_stderr(stderr)
    stdout.lines.last.split(':', 2).last.strip == 'OK'
  end

  def raise_on_stderr(stderr)
    clean_stderr = stderr.lines.reject { |line| line =~ /\ALibClamAV Warning:/ }
    return if clean_stderr.empty?

    fail RuntimeError, "The 'clamscan' command line was unable to execute. #{clean_stderr.join("\n")}"
  end

end
{% endhighlight %}

Doesn't that look a lot better? We did quite a bunch of changes here.

* Comparing the output of parse_clamscan_output to a string is not very descriptive, and it's brittle. It's better to just have a method that changes the output to the direct answer we are interested in.
* Move exception handling into its own method
* [Prefering `fail` over `raise` for original errors](http://devblog.avdi.org/2014/05/21/jim-weirich-on-exceptions/)
* Use [String#lines](http://apidock.com/ruby/String/lines) instead of [String#split](http://apidock.com/ruby/String/split)
* Extract the command variable to a method
* Skip unnecessary cast from Array to String and use [Array#empty?](http://apidock.com/ruby/Array/empty%3F)

Only one more line to go. There is still a train wreck in the ```clamscan_result_safe?``` method.

{% highlight ruby %}
def clamscan_result_safe?(stdout, stderr, status)
  raise_on_stderr(stderr)

  result_line = stdout.lines.last.strip
  scanned_file, result = result_line.split(':' 2)
  result == 'OK'
end
{% endhighlight %}

That's a lot easier to understand. 🎓

This improvement also hints at an other improvement I could add: By making sure the ```scanned_file``` actually matches the ```@file_path``` given, we would be even more sure about the behavior of this class.

# Final thoughts

There is still room for improvement here.

* Better handling of the clamscan binary, and PATH
* Extracting the clamscan output processing to it’s own object?
* Does it make sense that calling a scanner returns false when infected?
* You see something I missed? 🔎

Let me know if you like more posts like this. Also, let me know if you are interested to seem me do this same refactoring using TDD. Or, if you have a piece of code you'd like me to refactor; send it to me! 📩

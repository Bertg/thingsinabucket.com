---
layout: post
title: Refactoring for fun and profit
categories: console
---

Today I will guide you through a refactoring a class used in one of the project I work on. Idealy we should be doing this test driven, but - as this is for fun - I won't today.

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

This is our a simple interface around the [clamscan]() scanner. This code has several smells.

* Turning of the virus scanner is done using a global variable
*
*

Our mayor goal is to keep backward compatibility with the rest of the code base, while cleaning up the code.

# \#1 Separate the generik VirusScanner from the specific clamscan.

This way I can focus on the the VirusScanner without needing to worry about Clamscan.

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
    return @scan if @scan

    command = Cocaine::CommandLine.new(
      'clamscan', "--no-summary :file"
    ).command(file: file_path)
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

Cool. I could even choose to first refactor Clamscan now if I wanted to.

# \#2 Removing the 'virusscan_off' setting

Next I want to the app specific 'virusscan_off' setting. Honestly I'm ashamed I ever wrote that :)

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

This is already a lot better. We created an easy to understand interface for VirusScanner's scanner. As the behaviour of the class is now configurable it become very easy to extract the 'virusscan_off' setting.

{% highlight ruby %}
scanner_with_off_switch = ->(*args) {
  Settings['virusscan_off'] ? false : ClamScan.new(scanner).infected?
}
{% endhighlight %}

However, this breaks my requirement of not having to change the rest of the code, and it requires me to know what the default scanner is.

Lets do better!

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

I prefer the later. There is also one mayor change in the behaviour of these 2 solutions.

When calling ```VirusScanner.new(path, scanner: ->(*) {}).infected?``` the solution using prepend will always return false; while the option using default_scanner will use the passed scanner.

Ok. I'm pretty happy with that. Let's focus on the ```ClamScan``` class.

# \#3 Remove not needed gem

[Cocaine] has a lot of powerfull features and has it use cases. But here it's purely used to safely use a file name. [Shellwords can deal with this easily].

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

Open3.popen3 opens a stdin pipe, we don't need that at all. A better choice would be [Open3.capture3].

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

Let us also clean up the situation a bit

{% highlight ruby %}
class ClamScan

  attr_reader :file_path

  def initialize(file_path)
    @file_path = file_path
  end

  def infected?
    @infected ||= begin
      stdout, stderr, status = Open3.capture3(command)
      clamscan_result_safe?(stdout, stderr, status)
    end
  end

private

  def command
    Shellwords.join(['clamscan', '--no-summary', file_path])
  end

  def clamscan_result_safe?(stdout, stderr, status)
    raise_on_stderr(stderr)
    stdout.lines.last.split(':', 2).last.strip != 'OK'
  end

  def raise_on_stderr(stderr)
    clean_stderr = stderr.lines.reject { |line| line =~ /\ALibClamAV Warning:/ }
    return unless clean_stderr.any?

    fail RuntimeError, "The 'clamscan' command line was unable to execute. #{clean_stderr.join("\n")}"
  end

end
{% endhighlight %}

We applied a bunch of changes

* Comparing the output of parse_clamscan_output to a string is not very descriptive, and even brittle. It's better to just have a method that changes the output to the direct answer we are intrested in.
* [Prefering `fail` over `raise` for original errors]()
* Use [String#lines]() instead of [String#split]()
* Extract the command variable to a method

Only one more line to go. There is a trainwreck in the ```clamscan_result_safe?``` method.

{% highlight ruby %}
def clamscan_result_safe?(stdout, stderr, status)
  raise_on_stderr(stderr)

  result_line = stdout.lines.last.strip
  scanned_file, result = result_line.split(':' 2)
  result != 'OK'
end
{% endhighlight %}

That's already a lot easier to understand. This also indicates a small improvement I could add. By making sure the scanned_file actually matches the @file_path given to us we would be even more sure about the behaviour of this class.

And finally Let us simplify the integration between both

{% highlight ruby %}
class ClamScan

  def self.call(*args)
    new(*args).infected?
  end

  attr_reader :file_path

  def initialize(file_path)
    @file_path = file_path
  end

  def infected?
    @infected ||= begin
      stdout, stderr, status = Open3.capture3(command)
      clamscan_result_safe?(stdout, stderr, status)
    end
  end

private

  def command
    Shellwords.join(['clamscan', '--no-summary', file_path])
  end

  def clamscan_result_safe?(stdout, stderr, status)
    raise_on_stderr(stderr)
    stdout.lines.last.split(':', 2).last.strip != 'OK'
  end

  def raise_on_stderr(stderr)
    clean_stderr = stderr.lines.reject { |line| line =~ /\ALibClamAV Warning:/ }
    return unless clean_stderr.any?

    fail RuntimeError, "The 'clamscan' command line was unable to execute. #{clean_stderr.join("\n")}"
  end

end

class VirusScanner

  def self.default_scanner
    @default_scanner ||= ClamScan
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

There is still room for improvement.

* Better handling of the clamscan binary, and PATH
* Extracting the clamscan output processing to it’s own object?
* Does it make sense that ClamScan.call returns false when infected?
* You see something I missed?

Let me know if you like this type of post. Also, let me know if you are interested to seem me do the exact same thing but then using TDD.

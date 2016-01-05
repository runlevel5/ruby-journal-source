---
layout: post
title: "Testing Whirlwind tour in Crystal lang"
date: 2015-11-27 00:06
comments: true
categories: crystal
tags: crystal, test
author: "Trung Lê"
---

{% img left /images/2015-11-27-testing-whirlwind-tour-in-crystal-lang/diamond-through-a-loupe.jpg %}

{{ post.title }}

TDD has been part of my development career thanks to Ruby. It is in my nature to try to figure out how to test code with a new language that I learn before picking up other concepts.

Part of my learning of [Crystal lang](http://crystal-lang.org), I am going to walk you through how to write test in build in testing DSL.

<!--more-->

## Prequisites

* Crystal 0.9.1
* Basic understanding how to write a test case

## Anatomy of a spec

Let's see one simple spec:

```
require "spec"
require "../lib/greeter" # my greeter class

describe Greeter do
  describe "#shout" do
    it "returns upcased string" do
      Greeter.new.shout('hello world').should eq "HELLO WORLD"
    end
  end

  describe ".hello" do
    it "returns a static Hello string" do
      Greeter.hello.should eq "Hello"
    end
  end
end
```

As you can see above that this crystal source file require [Spec](http://crystal-lang.org/api/Spec.html) module. A spec starts with a `describe` DSL, the parameter of the describe could be a class object or a string. In fact, it is recommended that you follow the convention of descring the classname follow by describing the method name. Just on the naming convention for method description, I observe that the `#` prefix is used to denote instance method and `.` prefix to denote class or module method. Spec main logic is wrapped under an `it` block. The matcher `should` divide the caller which is on the left hand side and the expected result on the right side. I'll run through few common matcher in Matcher section.

If you are familiar with RSpec, you could clearly see the similarity between RSpec and Crystal's Spec, I am sure that Crystal is heavily inspired by the RSpec syntax.

By convention, spec file lives under `spec/` folder.

## Run a spec

To run a spec, simply run `crystal spec <spec_path>`, for eg:

```
crystal spec spec/my_spec.cr
```

Furthermore, you could run specific test by providing the line number that the spec case is on, for example:

```
crystal spec spec/my_spec.cr:10
```

This must be one of the best feature I enjoy with RSpec in Ruby world and I am glad that Crystal implemented it out of the box. I wish that the default Minitest that come with Ruby support this feature too.

## Structure a test suite

A test suite by convention consists of many spec files that resides under `spec` folder. Spec files must comply to `_spec.cr` suffix, eg: `spec/greeter_spec.cr`.

Crystal CLI `crystal spec` (without specifying the spec file) runs the whole test suite by recursively run all `spec/**/*_spec.cr`.

### Callbacks/Hooks

Spec fixture should be setup with `Spec.before_each` callback and then tear down with `Spec.after_each` callback. For example:

```
describe FetchFilm do
  describe "#fetch" do
    Spec.before_each do
      @@film_title = 'Hero'
    end

    Spec.before_each do
      @@film_title = nil
    end

    context "when title is provided" do
      it "should fetch correct movie" do
        FetchFilm.new.fetch(@@film_title).should be_a(Film)
      end
    end
  end
end
```

I try to look for equivalent of `let` and `let!` in Crystal but I could not any. So if you want to share a fixture, it'd be best to make it a global instance.

### Spec helper

I recommend creating a `spec/spec_helper.cr` where common spec helpers are shared between specs. At its simplest form, it could just have:

```
require "spec"

Spec.before_each do
  # things you want to run before every spec
end
```

and every spec should include this helper:

```
require "./spec_helper"
require "../lib/greeter" # my greeter class

describe Greeter do
  # same as above example
end
```

### Load file when required

Please avoid loading every library files in spec helper like this:

```
require "spec"
require "../lib/*"
```

To me this is an anti-pattern, autoloading all only slows down your test suite. Instead you should load the source on demand in the spec file:

```
require "./spec_helper"
require "../lib/file_i_want_test"

describe "BlahBlahBlah" do
  # test code here
end
```

## Expectation matcher

There are a set of matchers defined in [Spec::Expectation](http://crystal-lang.org/api/Spec/Expectations.html). Matchers must be called after object extension syntax `should` or `should_not`.

The most common matcher is `eq` that test if 2 object have same value:

```
a.should eq(b)
```

Or you care about the identity of the object, you can test if both share same object_id with `be` matcher:

```
a.should be(b)
```

And nothing guarantees that an object can be nil, we have `be_nil` matcher to test the nil-ness:

```
nil.should be_nil
```

Well, if you want to test a boolean? there are `be_false` and `be_true` matcher:

```
true.should be_true
false.should be_false
```

There are also matcher to test Truthy/Falsey `be_truthy` and `be_falsey`:

```
false.should be_falsey
nil.should be_falsey
"anthing other than false or nil".should be_truthy
```

Not to mention, we could test inclusion with matcher `contain`:

```
[1,2,3].should contain(1)
```

and have you used the unpopular `be_close` matcher to test if the actual result is equal to expected result plus/minus delta:

```
5.2.should be_close(5, 0.5)
```

Still want more? regex fan must be raved about `match` matcher:

```
"troll".should match /troll/
```

Or you might be wondering how to test the type of an object? look no further than `be_a` matcher:

```
"troll".should be_a(String)
```

Last is the `expect_raises` macro to test a block invocation would raise exception or not:

```
expect_raises(Exception) do
  raise "Error"
end

expect_raises(Exception, "Error") do
  raise "Error"
end

# you can even parse in regex to match the exception message
expect_raises(Exception, /Err/) do
  raise "Error"
end
```

As of version 0.9.1, `Spec::DSL` is not very well documented, however the source is very straight-forward to pick up. I am not going to into details how to write a custom matcher, let's leave it as a topic for the future writing.

## Extra DSL for better spec writing

### Provide more context with `context` DSL

For most of the cases, you want to provide more context for your spec, you can always use `describe` for the job like Ruby's Minitest or you can use `context` DSL:

```
describe String do
  describe "#to_s" do
    context "when value is nil" do
      it "should return empty string" do
        nil.to_s.should eq("")
      end
    end
  end
end
```

### Fail fast

Spec::DSL provides `fail` DSL for failing a spec fast:

```
describe String do
  describe "#to_s" do
    context "when value is nil" do
      it "should return empty string" do
        fail "I want it to fail on purpose"
      end
    end
  end
end
```

which would yield `Failure/Error: fail` when the spec is run.

### Mark spec pending

Spec can be marked pending with `pending` DSL. It is usually used for quick interface design in TDD style:

```
describe String do
  describe "#to_s" do
    context "when value is nil" do
      pending "should return empty string" do
        #Dude, can you please implement it
      end
    end
  end
end
```

Please be noted that it replaces the `it` DSL rather than living under the `it` block.

### Custom assertion

It is not always that you could test complex logic with default expectation matcher. Do you know that you could define custom assertion with `assert` syntax? The main assertion logic resides as a block:

```
describe String do
  describe "#to_s" do
    context "when value is nil" do
      assert "should return empty string" do
        nil.to_s == ""
      end
    end
  end
end
```

## Conclusion

Crystal language comes with a full-feature Spec module. The spec follows RSpec-like format with `describe` and `it`. Besides, the availability of popular matchers and setup/tear down callbacks makes testing working very pleasant. I've yet to figure out how Crystal do mock/stub.

---
layout: post
title: "minitest let() is lazy"
date: 2013-04-13 10:31
comments: true
categories:
tags: testing, ruby, rails, minitest
author: "Trung Lê"
---

{{ post.title }}

Don't you know that minitest also have a similar `let()` helper that does exactly the same job as the one of RSpec. But some people do not fully understand the difference between defining object in `let()` and `before()` block.

<!--more-->

So you've been doing TDD, that is writing tests. Tests is like app documentation, it needs attention as much as the main code does. One good way to refactor your test is to use `let()`.

We have a simple spec below:


```ruby
describe Post do
  before do
    @post = Post.create!(name: 'Awesome Post')
  end

  describe '#publish' do
    it 'should publish to the world wide web' do
      @post.publish
      @post.published.must_equal true
    end
  end
end
```

that could be refactored to:

```ruby
describe Post do
  let(:post) { Post.create!(name: 'Awesome Post') }

  describe '#publish' do
    it 'should publish to the world wide web' do
      post.publish
      post.published.must_equal true
    end
  end
end
```

Our test is now easier to read. Some developers naively believes the `let()` just provides a syntax sugar to the test, in fact it is totally different from the block way. It's lazy loaded. See the code below:

```
let(:post) { Post.create!(name: 'Awesome Post') }

Post.count
# 0

post

Post.count
# 1
```

the `post` is not created until it is invoked. This definitely helps improve the speed of test in which object is initialized upon being referenced. Yet a caveat for tests on collection of objects that does not reference the object directly in the test. For example:

```ruby
describe Post do
  let(:post1) { Post.create!(name: 'Awesome Post') }
  let(:post2) { Post.create!(name: 'Awesome Post') }

  describe '.bulk_publish' do
    it 'should bulk publish to the world wide web' do
      Post.bulk_publish
      post1.published.must_equal true
      post2.published.must_equal true
    end
  end
end
```

To some people surprise, the test will fail because we do not invoke `post1` and `post2` in the test, thus 2 posts are not created before `Post.bulk_publish` is invoked. You tap those 2 objects before the test with:

```ruby
describe '.bulk_publish' do
  before do
    post1
    post2
  end
  ...
end
```

Which I find very ugly. In such cases, it'd be best interest to use the block way to define objects.

```ruby
describe Post do
  describe '.bulk_publish' do
    before do
      @post1 = Post.create!(name: 'Awesome Post')
      @post2 = Post.create!(name: 'Awesome Post')
    end

    it 'should bulk publish to the world wide web' do
      Post.bulk_publish
      @post1.published.must_equal true
      @post2.published.must_equal true
    end
  end
end
```

So now you know the behaviour of `let()`, I highly recommend you go through your test and apply what you have learned. See you guys in the next tutorial.
---
layout: post
title: "A simple shift cipher with Ruby"
date: 2014-04-27 01:28
comments: true
tags: ruby, crytography
author: "Trung Lê"
---

{{ post.title }}

Crytography was born thousands years ago, dating back to early day of the Roman which Ceasar
used a very simple form of substitution cipher to encrypt military message. Nowaday cryptography
is much more complex and virtually not solveable in some cases (well until quantum computing).

Today I am going to show how to implement the [Caesar's Cipher](http://en.wikipedia.org/wiki/Caesar_cipher)
with improvements:

* Based on Unicode table
* Accept case sensitive

Read on you'll be surprised how easy it is with Ruby!

<!--more-->

## Concept

The concept is pretty much the same with Caesar's Cipher in which character is shifted
by a key to the left to generate encrypted text. You can read more about it on WWW.

## Implementation

Unicode characters are mapped to Unicode table and are represented with
number in bytes.

English alphabets are basic Latin which are mapped in range 32 -> 127 on Unicode table.
See the [table](http://unicode-table.com/en/#basic-latin)

The trick is to convert the message to digit so we could shift the
position of each character.

Ruby comes with method `String#bytes` that yields the bytecode of the string, eg:

```ruby
"Super Secret".bytes
#=> [83, 117, 112, 101, 114, 32, 83, 101, 99, 114, 101, 116]
```

And to get back our original text, we use `String#pack`:

```ruby
[83, 117, 112, 101, 114, 32, 83, 101, 99, 114, 101, 116].pack('U*')
#=> "Super Secret"
```

The argument 'U*' means unicode. Google if you want to know other arguments
for this method.

All the hard work! Ruby has made hard things trivial.

### Encryption

Now we could implement our encrytion easily by shifting the byte number
to the right by a _key_.

```ruby
module ShiftCipher

  LATIN_RANGE = (32..127).to_a.freeze

  def self.encrypt(message, shift_key: 3)
    to_string(shift_bytes(message.bytes, shift_key))
  end

  private

  def self.shift_bytes(bytes, shift_key)
    bytes.map { |i| i + shift_key }
  end

  def seflf.shift_byte(byte, shift_key)
    while (bye + shift_key) > LATIN_RANGE.last
      i = ((bye + shift_key) - LATIN_RANGE.last) - 1

    end
  end

  def self.to_string(bytes)
    bytes.pack('U*')
  end
end
```


33 - 3 -32

-2


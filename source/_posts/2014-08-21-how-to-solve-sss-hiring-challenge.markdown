---
layout: post
title: "How to solve Silicon Straits Saigon hiring challenge"
date: 2014-08-21 17:04
comments: true
categories:
tags: hack, fun
author: "Trung Lê"
---

Hello folks

For whom who might have not heard of Silicon Straits Saigon (SSS), this
company is one cool company in Vietnam with strong focus in tech such
as iOS and Web. When I use the word cool, I did not mean it before I
touch base with their hiring challenge.

SSS is known to have a quite unique way to challenge potential employees.
They give you a very cryptic page [http://hiring.siliconstraits.vn](http://hiring.siliconstraits.vn) and
asks you to hack them.

Well, today I am going to show you how to solve this problem step by step.
I hope Mr. An (Director of SSS) won't hate me for this.

Spoiler alert! Go give the challange a go yourself before reading!

<!--more-->

## The challenge

Use your favourite browser and go apply for job at SSS at http://hiring.siliconstraits.vn.

Okay, where is my form? How could I fill in my name and my CV. Is this some kind
of troll? Oh hold on, I see something cryptic, that is cryptic `L3VzZXJzLzppZC9hcHBseSAg` string.

And the problem is "Call the API". Which API? Give me API, damm it!

## Decode the Engima (not really)

Honestly, I am clueless, been asked to call an API and given an encrypted string. I tried
to squeeze my tiny brain to put these two pieces of the puzzle together, and the only hypothesis
that is plausible enough is that the encrypted string has all details about the API.

Firstly, we need to decode this encrypted string. As you might know there are various ways
to encrypt a message, and you could go easy with Caesar Cipher or maniac with AES-256bit.
If you think SSS would give you an AES-256bit encrypted message, then I might be looking
for job at wrong place. SSS touts themselves a iOS and Web firm, not a security firm, thus
pragmatically I would not want to go back opening thick books of algorithm which turns
out useless because in theory you could not decode it within your lifetime!

Now, I assume this is something that does not require a private key to decrypt. Something
that is easy, something that is related to SSS's expertise that is iOS or Web. Ouch, my
brain cell is burning, it hurts, by instinct I know that Web CSS and Backend code uses
Base64 quite often, so it is only plausible that I give Base64 a trial.

Let's fire up Ruby REPL, that is `irb`:

```
$ irb
> require 'base64'
> Base64.decode64 "L3VzZXJzLzppZC9hcHBseSAg"
> => "/users/:id/apply  "
```

Wow, just by chance, I got it right the first time ;). So we have another clue, a string
of "/users/:id/apply". So this must be the location of the HTTP API that SSS call to

## Decode the :id param

Based on my experiences, this route "/users/:id/apply" looks like a RESTful resource
of user which takes in a param `:id`. But what's the `:id`? You asked me? I have no clues!

Because this is a HTTP Web API, let's divert from the question and play around by calling
to this Web services first.

I used `curl` to make all calls. I chose `id=1` for the API call:

```
$ curl -X GET http://hiring.siliconstraits.vn/users/1/apply
{"error":"Please authenticate using Basic Auth with your token as password"}
```

The response gives me another clue, that is the service uses Basic Auth with token
as the password. This clue is very confusing because it does not tell me if my
param `:id` is correct or not.

I spent the next few minutes trying out other hypthosesis.

With the assumption this is a User resource, by convention I could get all
users from SSS by calling GET request to `/users`. I might give me the list
of `:id` that might be the key for the problem.

```
$ curl -X GET http://hiring.siliconstraits.vn/users
Good luck, keep trying ;)
```

Sadly, it turns out that SSS would not give it to you that easily. Who would anyway right?
If SSS does, I would stop here and look for a job elswhere.

Okay, next let's try to call a POST on the same URL:

```
$ curl -X POST http://hiring.siliconstraits.vn/users
{"error":"email is required (prefer Gravatar)"}
```

This time the message is helpful, it asks me to post a param `:email`. So I did:

```
$ curl -X POST -d "email=trung.le@ruby-journal.com" http://hiring.siliconstraits.vn/users
{"email":"trung.le@ruby-journal.com","id":3615,"mobile":null,"name":null,"profile_url":null,"token":"AQGNAOFHQFSECGD","updated_at":"2014-08-21T07:01:48Z"}
```

Viola! The response gave me the `:id` plus the `:token`

## By passing the Auth

Now we have 3 pieces:

* URL to make a call to (I assume a POST call) is: http://hiring.siliconstraits.vn/users/:id/apply
* `:id` is `3615`
* `:token` is `AQGNAOFHQFSECGD`

Let's make a last call to the API and hope that SSS would come back or I would come to see them
directly (still need to sort out few details for the Ruby Meetup 2 in September though). In the last
section, we know that SSS is using Basic Auth and token.

With the knowledge of the Auth type and token, we could send this detail to SSS for auth by setting
the header with `Authorization: Basic <TOKEN>`:

```
$ curl -X POST -H "Authorization: Basic AQGNAOFHQFSECGD" http://hiring.siliconstraits.vn/users/3615/apply
{"error":"Please authenticate using Basic Auth with your token as password"}
```

WTF?! It does not work!? I thought that it would be similar like Spotify API or Twitter API. So I tried
again with different header:

```
$ curl -X POST -H "Authorization: Token token=AQGNAOFHQFSECGD" http://hiring.siliconstraits.vn/users/3615/apply
{"error":"Please authenticate using Basic Auth with your token as password"}
```

Still not working! Hmm, so it seems to me my approach is wrong. I took a step back and thinking about the
word 'Basic Auth'. As you might have know, there is a more primitive HTTP Basic Authentication out there
in the wild, so it might be that. Now try again:

```
$ curl --user 3615:AQGNAOFHQFSECGD -X POST http://hiring.siliconstraits.vn/users/3615/apply
{"error":"name, profile_url, mobile are required. Please update before apply (provide real, public data please)"}
```

OMG, it worked (well partially), I was right!

FYI, the `--user` param of curl takes `user:password`.

We now need to update the details  `name`, `profile_url` and `mobile` and come back to submit :)

## Update missing details

Okay, so I was right, this is a REST API, which means by convention, sending a PUT request to `/users/:id`
would upate the details for me.


```
$ curl --user 3615:AQGNAOFHQFSECGD -X PUT -d name="Trung Le" -d profile_url="http://github.com/joneslee85" -d mobile="1080 888 888 888 888" http://hiring.siliconstraits.vn/users/3615
{"email":"trung.le@ruby-journal.com","id":3615,"mobile":"1080 888 888 888 888","name":"Trung Le","profile_url":"http://github.com/joneslee85","token":"AQGNAOFHQFSECGD","updated_at":"2014-08-21T08:09:27Z"}
```

Hey hey, the response indicated that my details are now full-filled!


## Connecting all the dots!

Cut it short to 1 line and so I could head to the GYM:

```
$ curl --user 3615:AQGNAOFHQFSECGD -X POST http://hiring.siliconstraits.vn/users/3615/apply
{"message":"Congratulations Trung Le!, here's the next step /users/:id/profile"}
```

Finally I got to the last step, hold on! WTF it is not finished?! Are you kidding me? Okay, let's
check out my profile. My advice is making sure you check your CV at least 4 times before scheduling
the interview.

```
$ curl --user 3615:AQGNAOFHQFSECGD -X GET http://hiring.siliconstraits.vn/users/3615/profile
Please authenticate yourself using token param
```

Boom! -_- it did not work like I expected. Let's give it the token param

```
$ curl -X GET "http://hiring.siliconstraits.vn/users/3615/profile -d token="DGCESFQHFOANGQA"
```

and the response was:

```
<!DOCTYPE html>
<html>
<head>
  <title>We're young & we're hiring</title>
  ...
</html>
```

The response is HTML with content about my profile and embedded within is another secret:

```
================================================
U3R1Y2s/IEhlcmUncyBhIHRpcDogYml0Lmx5L1VUdlpoMyAg
================================================
```

and it is also Base64 encrypted string, decoding it gives:

```
Stuck? Here's a tip: bit.ly/UTvZh3
```

It turns out it is in turn encrypted with QR code. I start to like this game and what I get is:

```
Don't stress, just REST
```

Very interesting, what is REST?! Nvm, lol

I guess this ends our fun. I hope SSS would come up with some more fun in the future.

Bye folks





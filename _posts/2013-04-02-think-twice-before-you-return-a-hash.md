---
layout: post
title: "Think twice before you return a hash"
description: ""
category: development
tags: [ruby, hashrocket]
---
{% include JB/setup %}

Recently I got the change to do an hour programming audition for the awesome
people at [hashrocket](http://www.hashrocket.com). I was excited, I brushed up on
the basics the night before, readied myself with some herbal tea and logged in
to the audition with enthusiasm.

What I found was 100 lines of difficult ruby code.  I started with what I would
normally do:

1. Break big methods into small methods.
2. See if any of those small methods give me a clue as to what responsibilities
(classes) should be extracted.
3. Look for switch statements or nested if/else statements and see if I can pull
some kind of strategy out.

```ruby
def foo
  "foo"
end
```



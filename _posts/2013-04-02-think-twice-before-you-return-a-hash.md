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

After about 20 minutes I had only refactored a couple of methods.  I was on the
spot and couldn't figure out what was wrong.  I kept gravitating towards the
the big method but didn't know how to break it apart.

Here is the code.

    module Domainatrix
      class DomainParser
        include Addressable

        attr_reader :public_suffixes

        def initialize(file_name)
          @public_suffixes = {}
          read_dat_file(file_name)
        end

        def read_dat_file(file_name)
          # If we're in 1.9, make sure we're opening it in UTF-8
          if RUBY_VERSION >= '1.9'
            dat_file = File.open(file_name, "r:UTF-8")
          else
            dat_file = File.open(file_name)
          end

          dat_file.each_line do |line|
            line = line.strip
            unless (line =~ /\/\//) || line.empty?
              parts = line.split(".").reverse

              sub_hash = @public_suffixes
              parts.each do |part|
                sub_hash = (sub_hash[part] ||= {})
              end
            end
          end
        end

        def parse(url)
          return {} unless url && url.strip != ''
          url = "http://#{url}" unless url[/:\/\//]
          uri = URI.parse(url)
          if uri.query
            path = "#{uri.path}?#{uri.query}"
          else
            path = uri.path
          end

          if uri.host == 'localhost'
            uri_hash = { :public_suffix => '', :domain => 'localhost', :subdomain => '' }
          else
            uri_hash = parse_domains_from_host(uri.host || uri.basename)
          end

          uri_hash.merge({
            :scheme => uri.scheme,
            :host   => uri.host,
            :path   => path,
            :url    => url
          })
        end

        def parse_domains_from_host(host)
          return {} unless host
          parts = host.split(".").reverse
          public_suffix = []
          domain = ""
          subdomains = []
          sub_hash = @public_suffixes

          parts.each_with_index do |part, i|
            sub_hash = sub_parts = sub_hash[part] || {}
            if sub_parts.has_key? "*"
              public_suffix << part
              public_suffix << parts[i+1]
              domain = parts[i+2]
              subdomains = parts.slice(i+3, parts.size) || []
              break
            elsif sub_parts.empty? || !sub_parts.has_key?(parts[i+1])
              public_suffix << part
              domain = parts[i+1]
              subdomains = parts.slice(i+2, parts.size) || []
              break
            else
              public_suffix << part
            end
          end

          {:public_suffix => public_suffix.reverse.join("."), :domain => domain, :subdomain => subdomains.reverse.join(".")}
        end
      end
    end

So much is wrong with this I don't even know where to start, but the next morning
I woke up with a clear head.  The author was returning a hash from both methods
that deal with parsing the domain.

If I had to do it over again I would immediately create a `struct` and start
testing and abstracting logic.  In case your not familiar a struct works like a
hash in ruby.

    Foo = Struct.new :bar, :baz
    foo = Foo.new "hello", "world"
    foo[:bar]  #=> "hello"
    foo["bar"] #=> "hello"
    foo.bar    #=> "hello"
    foo.baz    #=> "world"

It can also take a block.

    Foo = Struct.new :first, :last do
      def full_name
        "#{first} #{last}"
      end
    end

    foo = Foo.new "Micah", "Woods"
    foo.full_name #=> "Micah Woods"

How does this help us?  Because a `struct` works just like a hash, we can convert
the return to the `struct` with minimal modifications to the existing system. Run
our tests, and start to extract logic.  This blog is getting a lot longer than I
intended so I will post an example of refactoring some of this logic soon.

Until then, remember, NEVER RETURN A HASH (at least not without real good reason)

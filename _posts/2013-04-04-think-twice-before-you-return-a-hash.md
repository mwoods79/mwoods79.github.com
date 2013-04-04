---
layout: post
title: "Think twice before you return a hash"
description: ""
category: development
tags: [ruby, hashrocket]
---
{% include JB/setup %}

Recently I got the chance to do an hour programming audition for the awesome
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
the big methods but didn't know how to break it apart.

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

So a lot is wrong with this code.  The most fundamentally flawed logic in this
code is not the fact that it has large methods, or a file parsing that obviously
does not belong but the conscious decision to return hash's from the methods.

When a hash is return from a method it is essentially an anonymous object.  It
has methods, and state, but no constructor and no class definition.  Think about
how much of this code would just vanish if an object was returned.

Since this file actually had other files that depended on this class returning
a hash, we will have to imagine some code.

    class ParsedDomain
      attr_writer :public_suffix, :subdomain
      attr_accessor :domain

      def initilize uri
        @uri = URI.parse uri
      end

      def public_suffix
        @public_suffix || ""
      end

      def subdomain
        @public_suffix || ""
      end

      def scheme
        @uri.scheme
      end

      def host
        @uri.host
      end

      def path
        return"#{@uri.path}?#{@uri.query}" if @uri.query
        @uri.path
      end

      def url
        @uri.url
      end

      def localhost?
        @uri.host == 'localhost'
      end
    end

(I could have done some cool meta programming here so that I didn't have to repeat
logic over and over again, but I wanted to be more explicit because this is an
example.)

Now lets look at what our original parse method *could* look like.


    def parse(url)
      return {} unless url && url.strip != ''
      url = "http://#{url}" unless url[/:\/\//]

      domain = ParsedDomain.new url

      return domain if domain.localhost?
      parse_domains_from_host domain
    end

    def parse_domain domain
      #... imagine code here to calculate domain
      domain.domain = domain
      domain.subdomain = subdomain
      domain.public_suffix = public_suffix
      domain
    end

Amazing how that would clean up.  The moral of the story is, if you see a method,
or write a method that returns a hash, you probably really wanted to return an
object (with a class you defined).  Take two steps back and re-evaluate what you
are doing.

I really enjoyed getting out of my comfort zone and trying to fix some nasty ruby
for a change.  I forget that code like this exists sometimes, and it is nice to
think about what makes code good for a change.

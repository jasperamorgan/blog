---
layout: post
title:  "Using Nix to run GitHub Pages & Jekyll"
date:   2024-02-12 03:09:19 +0000
categories: jekyll update
---

# Introduction

Gone are the days when one would setup and configure a machine by hand. We are in the era of configuration-as-code. This is not just for servers or cloud deployments but also applies to desktop deployments.

One of the downsides to GitHub Pages is that Jekyll requires me to configure a Ruby environment. Each time I have to do this it feels like I'm polluting my machine. (I'm not a fan of Ruby and I wouldn't use it for anything other than Jekyll.)

The good news is that with [Nix package manager](https://nixos.org/) I now have my Jekyll environment ready to go with one simple command. What's even better is that this is all done via configuration. This allows me to have the same Jekyll setup on any of my computers by running one command.


## Nix

There are many sources for learning about Nix and this isn't the place to find out what Nix is or how it works. If you haven't come across Nix before, I suggest you try the [First Steps](https://nix.dev/tutorials/first-steps/) tutorial.

## Nix and Ruby

I'm not going to pretend to know a lot about Ruby (or even pretend that I care).  Maybe all you need to know for this post are these three things:

1/ RubyGems is the way Ruby libraries are distributed and installed. The RubyGems configuration file is `Gemfile`.

2/ Ruby comes with a tool called `bundler` which manages the dependencies defined in the `Gemfile` (i.e. downloading the correct Gem versions).

3/ [bundix](https://github.com/nix-community/bundix?tab=readme-ov-file) is a community written util that is an nix-specific implementation of Ruby's `bundler`. They key point is that it uses the `nix store` for checking already downloaded Gems or downloading them based on a specific Gem version.

TL;DR - we are putting the Ruby libraries needed for running our GitHub Pages site in the `nix store`.

# Creating a Nix config for GitHub Pages

Let me first state that I followed the approach by [Nathan Bijnens](https://nathan.gs/2019/04/19/using-jekyll-and-nix-to-blog/) which is based on [Stefan Siegl's](https://stesie.github.io/2016/08/nixos-github-pages-env) writings. Please read and follow these great posts - there is no point in me repeating everything here.

To add to these great posts, let's add a little explanation about what's going on.

## Step 1 - Get all the Ruby dependenies

Firstly we need the RubyGem dependencies for our GitHub Pages site. This is achieved via the `bundix` util mentioned above.

I followed Nathan's approach and created a shell script to run the necessary commands to do this. I called mine `run-bundix.sh`.

## Step 2 - Run nix-shell when you want to start the Jekyll server

If you added the `default.nix` as described in the posts above, you just start the Jekyll server by starting `nix-shell`. Nice and easy!

# Fixes

I came across two problems running my GitHub Pages site using this approach.

##  nokogiri hash mismatch

`bundix` wasn't successful in handling the download of the `nokogiri` XML/HTML Ruby library. I saw an error similar to this:

```
hash mismatch in fixed-output derivation '/nix/store/a35hgacdgwh9a52mx6ssf0ijb60axkcn-nokogiri-1.11.7.gem':
```

The issue is that the Ruby bundler tries to optimise and request a platform specific version of the `nokogiri` library - in my case the Linux version. However, `bundix` calculates the hash used with Nix based on the non-platform version of the `nokogiri` RubyGem.

The fix is to tell the Ruby bundler not to use platform specific Gems. To do this, I updated `run-bundix.sh` (see Step 1 above) to include this bundler config: `bundler config set --local force_ruby_platform true`

Now `run-bundix.sh` looks like this:

```
#!/usr/bin/env bash

nix-shell -p bundler -p bundix --run 'bundler update; bundler config set --local force_ruby_platform true;  bundler lock; bundler package --no-install --path vendor; bundix; rm -rf vendor'

echo "You can now run nix-shell"
```

See https://github.com/nix-community/bundix/issues/88

## Jekyll error - webrick missing

The second error I encountered was this:

```
[...]/jekyll/commands/serve/servlet.rb:3:in `require': cannot load such file -- webrick (LoadError)
```

The solution was to explicitly declare the `webrick` dependency in the Gemfile by adding the line `gem "webrick"`.

Run the `run-bundix.sh` again to get the `webrick` library into the nix store.

See https://github.com/jekyll/jekyll/issues/8523

# Other notes

## Exclude Nix files from generated site

To prevent the nix specific files from ending up in the generated site, add the folloring to the bottom of the `_config.yaml`.

```
exclude:
    - run-bundix.sh
    - gemset.nix
    - default.nix
```

## Regenerate RubyGem dependencies

If you need to clean up and run `bundix` again from scratch delete the following files and run `run-bundix.sh` again.

```
Gemfile.lock
gemset.nix
```

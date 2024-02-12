---
layout: post
title:  "Using Nix to run Jekyll"
date:   2024-02-12 03:09:19 +0000
categories: jekyll update
---

Based on the approach from: https://nathan.gs/2019/04/19/using-jekyll-and-nix-to-blog/

Came across 2 issues.

1/ nokogiri hash mismatch

Fix - update setup-nix.sh with

`bundler config set --local force_ruby_platform true`

See https://github.com/nix-community/bundix/issues/88

2/ Jekyll error

Add `gem "webrick"` to your Gemfile

See https://github.com/jekyll/jekyll/issues/8523

Other

Prevent Gemfile, Gemfile.lock *.nix from ending up on the generated site.

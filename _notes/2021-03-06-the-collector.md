---
title: 'The Collector: Data-hoarding Webpages'
feed: show
---

# Introduction

Collecting HTML files to build a knowledge base.

## Sources

### Third-Party Sources

- [ ] Firefox bookmarks
- [ ] Pocket links
- [ ] Hacker news likes
- [ ] Reddit upvotes
- [ ] Feedly saved links
- [ ] Youtube likes

These are all third party services that we want integrations for.
We want to bring data from each of these sources into our main feed, so that:
  - we can apply filters to these feeds and take actions on the posts
  - we can consume these feeds in a common way
  - if we save the same article in reddit and hacker new, we dont want it to come
    up twice

### Firefox bookmarks

Firefox is a cross-device browser and has sync built in.  For most use cases we
are interested in, we either have firefox open (on desktop) or handy via share
menu (mobile).

We can hook into Firefox sync and retrive our bookmarks.

### Web Extension: Singlefile

This extension saves the current page as a single HTML file, with assets already
packed in. HTML files are a commonly accepted standard, and additionally, some
bookmarks are from sites we are logged into.  By downloading the assets and the
page as it's currently rendered, we can save content that would otherwise be
hidden behind authentication.

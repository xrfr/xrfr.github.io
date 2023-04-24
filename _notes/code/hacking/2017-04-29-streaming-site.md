---
title: Reverse-engineering a Streaming Site
feed: show
tags: hacking
---

### Table of Contents
1. [Introduction](#intro)
1. [Reverse Engineering](#reverse)
1. [Data Collection](#data)

<span id="intro"></span>
### Introduction

Streaming sites are plagued with ads, wait screens, and non-intuitive UI. Wouldn't it great if
we could just have the videos to watch at our leisure? Here we try to do just that.

<span id="reverse"></span>
### Reverse Engineering

First, we use our browser to inspect the site Looking through the DOM. There's a number of javacript
files the page requests, but only one that comes from it's own domain called `all.js`. We use `wget` to
download this file. The file looks like this. Each line contains some obfuscated javascript.

```javascript
eval(function(p,a,c,k,e,r){...
eval(function(p,a,c,k,e,r){...
eval(function(p,a,c,k,e,r){...
eval(function(p,a,c,k,e,r){...
eval(function(p,a,c,k,e,r){...
...// 19 total lines of this
```

This javascript was probably made using this popular [javascript
obfuscator](http://dean.edwards.name/packer/) called packer. One way we can get around this is by
replacing the 'eval' function in each line with `console.log`. This has the effect of undoing the
work that the obfuscator did, and instead of executing the javascript code, printing it out to
stdout.

```javascript
!function(t){var e="object",n="document",r="jQuery requires a window with a "...
!function(t){var e="undefined",i="Bootstrap's JavaScript requires jQuery",s=a...
!function(e){var a="floor",r="loop",t="onAutoplay",s="autoplay",i="target",n=...
!function(t){var e="function",r="Cannot find module '",n="'",i="code",o="MOD"...
!function(t){var e="init",i="prototype",s="options",n='[data-dismiss="modal"'...
!function(e){var n="init",t="prototype",o="options",a="extend",s="data",i="o"...
!function(t){var e="1.2.10",i="<div>",o='<div class="cluetip-outer">',s='<di'...
!function(T){var e="toLowerCase",S="length",i="call",o="i",P="",a="\\biPhone"...
!function(n){var e='[data-toggle="dropdown"]',o=".disabled, :disabled",t=".d"...
!function(t){var e="function",s="jquery",o="object",a="create",n="prototype",...
!function(t){var e="data",s="bs.toggle",o="object",i="string",n="options",l=a...
!function(t){var e="json",i="extend",n="ActiveHtml",o="string",a="create",s=v...
!function(t){var e="alert",i="warning",s="animated fadeInUp",o="animated fad"...
!function(e){var i="prototype",t=".swiper-container",o=".swiper-button-next",...
!function(t){var e="find",s="button",i="input",n="create",o="Movie.SearchAut"...
!function(t){var i="prototype",n="find",s="span",e="data",o="keep",_="name",h...
!function(e){var t="prototype",i="find",r="#player",s="data",a="id",o="#serv"...
!function(t){var s="prototype",a="star fa fa-star none",i="star fa fa-star-h"...
!function(t){var i="prototype",e="#movie",s="data",o="id",r="find",a=".episo"...
```

Now that looks better. But not very readable. We can use the npm module `js-beautify` to make this
easier to read. For convinience, I made each line into it's own seperate file. 

If we take a peek into the network requests that the page is making using chrome dev tools, we
notice calls to an endpoint called `ajax/episode/info`. Interesting... Let's see if we can find that
call in this code using `grep`. 

```bash
[player@macbook processed]$ ls
xaa.2.js	xad.2.js	xag.2.js	xaj.2.js	xam.2.js	xap.2.js	xas.2.js
xab.2.js	xae.2.js	xah.2.js	xak.2.js	xan.2.js	xaq.2.js
xac.2.js	xaf.2.js	xai.2.js	xal.2.js	xao.2.js	xar.2.js
[player@macbook processed]$ grep -rn --color "ajax/episode/info"
xaq.2.js:87:        Ne = "ajax/episode/info",
```

Beautiful. These are lines around line 87. Looks like the obfuscator is mapping strings to these
variables.

```javascript
Ae = "abort",
Ne = "ajax/episode/info",
We = "update",
De = "error",
_e = "Have an error occured, please refresh this page (ERR 1)",
Le = "direct",
Je = "iframe",
Me = "target",
Oe = "Server error, please refresh this page and try again",
He = "<iframe />",
```
Let's look for the the next instance of `Ne`. This is what we find. Looks like we're on the right track.

```javascript
t.episodeXHR = $.ajax(Ne, {
  data: {
      id: e[s](a),
      update: e[s](We) ? 1 : 0
  },
  success: function(i) {
      i[De] ? 
        t.showError(_e) : 
        i[x] === Le ? 
          t.apiXHR = t.loadApi(e, i) : 
          i[x] === Je && t.renderIframe(i[Me])
  },
  error: function() {
      console.log('uh oh');
      t.showError(Oe)
  },
  complete: function() {
      console.log('complete');
      e[s](ne, !1)
  }
})
```

Each video on the site has an id, it looks like that is what they're using to make this request.

If we look at the success handler, we see that first they check for the existance of a key `De` in
the response. If we take a peek at our block above, this translates to "error". If that key is
there, they call `showError`. Otherwise, if the property `x` (translates to "type"), is equal to
`Le` (translates to "direct"), we call this function `loadApi` with the contents of the first
response. If we search for that function in this file...

```javascript
loadApi: function(t, i) {
    var r = this,
        a = i.params;
    return t[s](Ze, t[s](Ze) || 0), $[ze](a, {
        _xtoken: e.XTOKEN,
        mobile: lr.phone() ? 1 : 0
    }), $.ajax(i.grabber, {
        data: a,
        success: function(e) {
            return;
            t[s](et, i.subtitle), e[De] ? e[De] === tt ? r.showError(it) : (r.reportError(t, e[De], e.token), r.handleEpisodeError(t, e[De])) : r.renderJWPlayer(t, e[s])
        },
        error: function() {
            t[s](Ze) < 1 ? (t[s](Ze, t[s](Ze) + 1), r.loadApi(t, i)) : r.showError(rt)
        }
    })
},
```

Looks like here, we take a part of the first response, "params", and use it to make this second
call to someplace `i.grabber`. After some fanagling, this is what I came up with.  

```javascript
const retrieve = (id) => {
  $.ajax('/ajax/episode/info', {
    data: {id: id, update: 0},
    success: (res) => {

      $.ajax('/grabber-api/', {
        data: res.params,
        success: res => {
          console.log(res);
        },
        error: console.log
      });
    }, error: console.log
  });
}
```

Running this on the site (using the console in chrome dev tools), using one of the ids from the
site, gives us what we were looking for, a set of hosted video links with content.  
(＾▽＾)

All we need now is a way to run this without a browser. [Zombie.js](http://zombie.js.org/) is quite amazing.

```javascript
var Browser = require('zombie');
var assert = require('assert');    

var browser = new Browser();
browser.visit(url, function () {    
  assert.ok(browser.success);

  browser.wait((window) => {
    return window.$ !== undefined;
  }, () => {
    console.log('browser loaded');
    const id = 'aks8ks';
    retrieve(id);
  });
});
```

<span id="data"></span>
### Data Collection

Now all we need is to collect these ids. After some more detective work on the website itself using
chrome dev tools, we find that ids come as links on the page itself for each anime. Let's use `wget`
to download their site recursively (following links on the site and downloading them).

```bash
[player@macbook ]$ wget -r \  # recursively download the website
                        -c \  # continue downloads if they were left partially downloaded
                        -nc \ # --no-clobber don't redownload pages if they already exist
                        "https://********/"
...
--2017-04-30 14:21:15--  https://********/genre/fantastic
Reusing existing connection to ********:443.
HTTP request sent, awaiting response... 503 Have you the ability to read quickly as the light ?
2017-04-30 14:21:15 ERROR 503: Have you the ability to read quickly as the light ?.

--2017-04-30 14:21:15--  https://********/genre/historical
Reusing existing connection to ********:443.
HTTP request sent, awaiting response... 403 You sent many requests in a short time, are you a bot ?
2017-04-30 14:21:16 ERROR 403: You sent many requests in a short time, are you a bot ?.
```

Oops, we forgot to play nice with the webserver and rate limit our requests. At least they have a
sense of humor. Another thing that was happening was the download for `robots.txt` was stalling out,
perhaps another countermeasure?

```bash
[player@macbook ]$ wget -r -c -nc \
                        -erobots=off \ # don't worry about getting robots.txt
                        -w 2 \         # wait 2 seconds between each request
                        "https://********/"
```

After a bit of progress, I notice that we start to download a bunch of page, one for each episide.
These all have the same links in them, so we don't need to get the duplicates. 

```bash
[player@macbook ]$ wget -r -c -nc -erobots=off -w 2 \
                        --reject-regex "********/watch/*/*" \ # watch urls for this regex and reject those pages
                        "https://********/"
```

I also notice that sometimes `wget` stops following links, seemingly in a random manner. I wanted to
get all the pages listed for each genre, and I could guess at how many pages there were total from the
website.

```bash
[player@macbook ]$ for i in {1..79}; do sleep 1; wget -c -nc -erobots=off "https://********/genre/action?page=$i"; done
```

+++
title = "Hugo"
date = "2019-03-06T00:57:32+05:30"
+++

I finally have a new post after about two years!  

This is the customary blog post where I talk about moving to the new static site generator.

Hugo.

This is way more flexible and easier to maintain than my previous setup, which was a hand-crafted website written
using vanilla HTML and CSS. After about 4 years I realized it was time to move on. It was a bit of a pain to add anything using raw HTML. A static site generator coupled with markdown for writing content was the obvious solution!  
Deciding which static site generator to choose was not an obvious choice however. I chose Hugo because it supports markdown, written in Go and is simple and lightweight.

But even then, it took me many weeks to completely move over to Hugo, with it requiring moving things to the new format,
obsessively tweaking everything and adding some content given my preoccupations.
I ended up editing the [Manis](https://github.com/yursan9/manis-hugo-theme), a minimalistic but elegant theme to fit my needs.

Another thing I should mention is that [xkcd-shuffle](/projects/xkcd-shuffle) now works again!  I had to rely on a Yahoo! API previously to wrap the xkcd endpoint with jsonp, as the xkcd API has neither jsonp nor does it have CORS headers, making it impossible to directly use it from within the browser. Unfortunately, YQL, the Yahoo! service I had used was discontinued. Fortunately, [someone](https://github.com/mrmartineau/xkcd-api) created a CORS enabled API for xkcd.
I don't know if anybody even uses xkcd-shuffle but hey it works now.

The last point also made me wonder if I should setup some form of a non intrusive tracking to get some very minimum of statistics about traffic on this site. I used to have Google Analytics at the very beginning, and although it was sometimes fun to look at, I took it down on account of principle. Given that I myself use a brwoser extension to block it on other places, it made no sense to keep it on my site.

Screenshots of the previous site
------------------------------
For posterity, here are some screenshots of the old site:

![old index image](/hugo/old-index.png)
<p style="text-align:center">Old home page</p>

![old projects image](/hugo/old-projects.png)
<p style="text-align:center">Old projects page</p>

That header image is basically a screenshot of xxd output of a screenshot of some code in my [SimpleNES](/projects/simplenes) project.

Now here's to hoping that this change would make me keep the site more updated and write more!


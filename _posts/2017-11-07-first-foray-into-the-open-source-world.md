---
layout: post
title: First Foray Into the Open Source World
published: true
date: '2017-11-07 08:46'

---

Delving into an open source project can seem daunting, but it’s a great way to get familiar with brownfield development while helping out a \*\*hopefully\*\* good cause.

Since I am a soon-to-be (just 2 more days!) graduate of [Turing](https://www.turing.io/), I’ll take all the experience I can working on codebases built by developers other than myself. That was the goal behind contributing to the [Open Food Foundation](https://openfoodnetwork.org/), a not-for-profit organization tasked with distributing food in a more streamlined and affordable way.

I started by going to the organization’s GitHub [repo](https://github.com/openfoodfoundation/openfoodnetwork) and looking through the list of issues.

<figure>
  <img alt="" src="/images/2017-11-07-first-foray-into-the-open-source-world/1*vPWzLzPx-qbl0EivEnvQzw.png" title="" />
  <figcaption>622 open issues…should be plenty to do&nbsp;here!</figcaption>
</figure>

Once I had an idea of the kinds of issues out there and felt capable of helping, I cloned down their repo and set out to get the code running in development.

I found the installation process fairly straightforward; there were a few hiccups that were immediately solved with a quick google search. The most unique part of this installation was creating a Postgres superuser before setting up the database. This was my first time doing so, but it was as easy as creating a table.

Once I had the code running in development I looked through it quickly to get a feel for its structure. With an idea of what was going on, I went back to the list of issues and found one I felt confident I could help with. I ended up selecting an issue where a save button was not selectable after deleting certain customer tags in the admin dashboard.

I set out to resolve the issue.

My first step was trying to replicate the issue. I navigated to the admin dashboard using the login information provided in the repo’s installation instructions, found the customers tab and tried to create a customer. I quickly realized that customers belonged to shops, and that there were fourteen shops to choose from. Hoping for the best, I selected the first shop and created a customer. I then added tags to that customer, saved them, and tried to delete them. So far no issues with the save button…damn. I proceeded to try creating customers for each shop, trying every permutation of edge case I could think of (leaving name blank, no billing address, multiple tags, only one tag, etc.). Unfortunately, try as I might, I couldn’t replicate the issue.

I went back to GitHub and let the people following this issue know that I could not reproduce it.

![](/images/2017-11-07-first-foray-into-the-open-source-world/1*DHl0RZn24XJUy8n8M_pH4A.png)

I then found out that the issue was happening in production/staging and was given more context. When I tried to use the admin credentials to access the dashboard in both production and staging, I found them to be invalid. It makes sense when I think about it, OFN wouldn’t just leave admin credentials out in the open for anyone to use. However, it means my ability to help on this issue was quite restricted.

While I was working on this issue I encountered two additional bugs. The first was a navigation bar style quirk where text became formatted strangely at certain resolutions. The other was in a pop-under notification that was supposed to list the number of unsaved changes but only ever listed “1”.

I wanted to bring light to these issues by adding them to the list on GitHub, but I didn’t want to create duplicated so I searched through the currently open issues to see if these had already been addressed. It turns out the first issue already had an open ticket, but the second didn’t, so I created a new issue.

![](/images/2017-11-07-first-foray-into-the-open-source-world/1*WacXnj1uiifqWqa0PjTjkQ.png)

I then tried to fix that very issue. A problem I encountered was the fact that the code was written in CoffeeScript for the Angular front-end. I was able to generally figure out what was happening, but I couldn’t find precisely what was causing the “unsaved changes” count to remain at “1”.

---

## What I Learned

Aside from having to familiarize myself with a large codebase I had never seen before, I found the most daunting challenge to be the testing suite.

I had never used PhantomJS, but implementing it was quite straightforward. However, the sheer size of the testing suite was something I found difficult to work with. I know of Zeus and its awesome ability to remove some of the pain points when dealing with vast amounts of tests. However, even Zeus needs to load your tests all the way through at least once, and OFN’s suite took multiple hours to run.

This made it very difficult when I tried to fix my third issue, which was to remove a potentially duplicate directive. To see how the change might affect the whole codebase, the testing suite would need to be fully run through twice to make sure nothing broke.

I ended up creating a PR for the issue I resolved, and am waiting to see if it gets accepted!

![](/images/2017-11-07-first-foray-into-the-open-source-world/1*OTq6HickLANR3h-W-1Cz_g.png)
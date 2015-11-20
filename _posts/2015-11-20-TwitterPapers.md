--- 
published: true
title: Mining Journal Article Recommendations from Twitter Feeds
layout: post
author: John
category: articles
tags: 
- projects
- Twitter

---

Twitter is great for learning about exciting new journal articles that are relavant to your field. I follow many scientists who are active on Twitter and have similar research interests to mine. The problem is that keeping up with my Twitter feed, and separating the signal from the noise, can be overwhelming. 

What I want is an automated way to mine my Twitter feed for journal articles. I also don't want to have to worry about hosting it myself, and I don't want to have to code a complicated solution if I can avoid it (I do enough of that at work). 

# A half-way solution

The simplest, but least elegant, solution for me was to create a Twitter list of all the scientists I want to follow for paper recommendations. I then added a column in [TweetDeck](https://tweetdeck.twitter.com) dedicated to that list, and I filtered it to only show tweets containing links. I tried to also filter the list to only see tweets containing "paper" OR "article," but there appears to be a bug in TweetDeck causing boolean operators not to work, at least in this circumstance.

# A better solution

I use RSS feeds heavily, so an ideal solution would convert my Twitter list to an RSS feed, then filter it to include only tweets containing links (and maybe even filter further to only include links to specific publisher websites). Unfortunately, I cannot find a service to convert a Twitter list to an RSS feed, but there are services that will convert individual user timelines to RSS feeds. This, combined with an RSS feed aggregator, yields a similar result.

Here's how to do it:

1. Use [TwitRSS.me](https://twitrss.me/) to create an RSS feed for each person you want to follow. Simply type their username in the text box, click "Fetch RSS," and copy the URL for the twitter feed. Even easier: the URLs are always of the form "https://twitrss.me/twitter_user_to_rss/?user={username}," so you don't even have to go to the website to do this. Just copy this template, go to step 2, and change {username} for each user you want to follow.
2. Use [FeedKiller](http://feedkiller.com/) to aggregate the individual RSS feeds into a single feed. Unfortunately I cannot find a way to edit an existing aggregate feed, so if you ever want to add a new user you have to build the list again from scratch. I simply save all the TwitRSS.me links to a text file on my computer and then just cut-and-paste them all into FeedKiller each time the list changes. Note that you have to specify the number of items you want to fetch from each feed, with a max of 10. Give your aggregate feed a title, then click "build it!" This is important: make note of the ID at the end of the feed. The actual URL of your aggregate RSS feed is: http://feedkiller.com/files/inc.xml.php?id={ID}.

The missing part of the solution is filtering the feed. I have heard that [FeedRinse](http://www.feedrinse.com/) is the best service for feed filtering, but it appears not to be working at the moment. They are promising a "2.0" upgrade at some point, but who knows when. I think that FeedRinse will also take care of Step 2 (i.e. it allows you to create an aggregate list and also filter it). I've signed up to be notified when the new FeedRinse is available, so I'll update this post if and when I get the full solution working.

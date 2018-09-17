---
layout: single
title: Projects
author_profile: true
permalink: /projects/
toc: true
toc_label: "Projects"
---

## [MIS BOT](https://github.com/ArionMiles/MIS-Bot)
Telegram bot to pull attendance from SJCET website, plus more features like bunk calculator, test reports, etc.

Originally created with the intent to help me avoid the defaulter's list due to low attendance in college, this project has grown to offer multiple services apart from just showing attendance. It's also capable of solving CAPTCHA on the college's login page. It's currently serving more than 200 students in my campus. It's one of my active projects, constantly undergoing development considering the feedback from the student users.

## [Torrent Automation](https://github.com/ArionMiles/Torrent-Automation)
My platform-agnostic torrent automation setup. Made with Filebot, qBittorrent, Pushbullet, and Telegram.
I use Filebot's AMC script for qBittorrent which automatically executes once a torrent has finished downloading, then the filebot script sorts them out into Movies/TV series and on the basis of that renames them into `SxxExx - Title` format and puts them into their respective `SeriesTitle/Season xx` folders.

For movies, it simply renames them to `MovieName (YEAR)` format and puts them in a folder of the same name.

Then I receive a Pushbullet/Telegram notification telling me if a Movie or a Series (with episode number and title) is ready and my Plex library is automatically updated.

## [Infinity Gauntlet]()
Started out as a fun little project me and my friend [Ujjwal]() decided to build seeing tensions rise in the subreddit /r/thanosdidnothingwrong as subscribers wanted half the population of the community to be banned, at random, emulating the "snap" from the movie Avengers: Infinity War and the mods were unable to follow through (really, it had ~120k users when we started working on this) as it wasn't feasible to manually ban 50% subscribers, given that reddit subscriptions are anonymous.
Our submission was among the many instrumental in bringing the reddit admins onboard and helping the community make it happen. You can read all about the event [here](https://redditblog.com/2018/07/12/thanosdidnothingwrong/).

Our solution works for the most part, scrapes usernames and bans them, but when you're mining data from a subreddit as popular and large as this, you quickly run through the Reddit's API rate limits which makes it a little slower. However, this was a nice project to interact with a community of fans and get them excited about an idea, which eventually became a reality, while at the same time we learnt about the official Reddit API's limitations and how using different source such as PushShift.io where it'd have been far more easier to get data from.

## [Zooqle-qBittorrent](https://github.com/ArionMiles/Zooqle-qBittorrent-Plugin)
A small search plugin for Zooqle for the qBittorrent client which allows searching for torrents from within the torrent client.
Also [included in the official search-plugins repository](https://github.com/qbittorrent/search-plugins), and installs with a single click.

## [Triton](https://github.com/ArionMiles/Triton)
A small Telegram Bot which fetches my github notifications directly to my Telegram client (I don't check emails that often). Deployed on Heroku and uses Github Public API v3.
---
title: Watching Spider-Man with Golang and AWS Lambda
date: 2021-12-25
type: "post"
description: "Reviving an old project for getting the best seats in the house"
tags:
  - serverless
  - go
---

I promise there are no spoilers in this blog, it's perfectly safe to read if you've not seen the movie yet.

Last weekend I saw the new Spider-Man ([respect the hyphen](https://www.reddit.com/r/RespectTheHyphen/)) movie No Way Home. Given that these movies are a big phenomenon, if you want to watch these opening weekend or the very first show, you need to book seats as early as possible or you can end up with having to settle for uncomfortable seats or worse, all shows sell out and you need to be careful avoiding spoilers on the internet.

I was sure that finding good seats was going to be next to impossible with this one, especially since theaters are operating at 50% Capacity due to COVID. So I dusted off good-old Diomedes.

# What is Diomedes?

For those of you just tuning in, Diomedes is one of my pet projects I wrote back in college to help me (and 200 random internet strangers) book tickets to Avengers Endgame. It does this by checking BookMyShow (BMS), which is a movie and events ticketing platform in India, similar to Fandango in the US.

Fun fact: I got the idea for the original Diomedes from [Mrinal](https://mrinalpurohit.in/)'s [moviealert](https://github.com/iammrinal0/django_moviealert) project!

You tell Diomedes your:

1. Movie's name,
2. The date for which you'd like to book tickets,
3. Language of the movie -- Lots of movies release in multiple languages
4. Dimension of the movie (2D/3D/etc.) -- One movie can be played in several different formats
5. Your favorite nearby movie theaters

and it continuously checks BMS. Whenever sales open for your specified alert, it sends you an email with the direct link to booking the tickets for all showtimes available.

What set Diomedes apart from other similar projects was that it didn't rely on webscraping, I instead wrote a light wrapper over their REST APIs they use for their Web App which made it more reliable, as webscraping is prone to failure when the markup changes (as it did with BMS).

BMS has a feature where you can mark yourself as "I'm Interested" for a particular movie and it lets you know when the movie is released, but it's not real time and it's a general reminder that the movie is "out", and not specific to your region or theaters.

## What happened to the original Diomedes?

Diomedes was originally deployed as a Django App in 2019 on a GCP VM. It was deployed as a docker compose app with

1. A Nginx reverse proxy
2. Postgres DB
3. Redis (abused as a queue)
4. Celery workers for cron jobs

I had turned off the Django app last year and let the domain expire, I didn't want to keep paying the recurring cloud bills and no one was really going out last year (lockdown) so it didn't make sense to keep it running.

That's the logical reason. The real reason is that the server I was running it on had run out of disk space due to what I can only attribute to the large log files my compose app created. I should've enabled rotating log files but lesson learnt, I guess?

Things were so messed up that even after we managed to clear up disk space, the docker daemon on my VM wouldn't work and I wasn't able to restart the compose app. [Namc](https://namc.in) helped me clear the disk space and fix my docker daemon (she's a wizard with containers).

However, the docker volume I was using for Postgres was still messed up and beyond recovery for reasons I cannot recall. As a result I lost all the existing user data. I don't care much about the data since I didn't plan on doing anything with it, but I still feel bad it happened. I felt it best to lay it down to rest at that point at revisit (or rewrite) it in the future.

### Lessons learnt?

1. Don't install docker using quick install scripts listed in the docs. Always install it via your package manager -- Namc
2. Never run databases in containers.

![We're Serverless people now](/images/serverless_people.jpeg "We're serverless people now")

## I'll make it Serverless

After a brief stint at [Antstack](antstack.io), I was sold on the simplicity of building with serverless compute and leveraging managed services to build applications.

So I thought to myself, let's rewrite the entire app using serverless offerings from AWS. That was an year ago now, and I have abandoned that project (more on why at a different time maybe).

I still needed something working for the upcoming movie, and I didn't want to use the Django App mainly for two reasons:

1. I didn't want to set up the django app. It's too much work (I could automate this with Terraform + Ansible but ain't nobody got time for that). Besides, using a VM to run the app costs money, and I'm a cheapskate.

2. I didn't want email notifications. With the way the Django app works, it's sent to address of the user setting the alert which would require everyone in our circle setting up an alert.

So now that I knew what I didn't want, I had some idea of what I did want.

1. Easy to setup infra for, and cheap.

2. Something that allows sending notifications to any interface I want.

I knew using AWS Lambda would make this 100% free since they have a generous free tier. I have also been working with Go at work recently, so I wanted to do this in Go this time around.

Fortunately, I had already done the heavy lifting in Go for this particular project 2 years ago, when I wrote [goBMS](https://github.com/ArionMiles/gobms/), which is a API wrapper for BookMyShow written in (you guessed it) Go.

So all I needed to make was

1. A function for searching the movie
2. A function for sending alert to an interface of my choice.

## The Architecture

Now that I had this, I created an architecture that looked like this:
![Diomedes-search architecture](/images/diomedes_search_arch.png "Diomedes-search Architecture")

There's five pieces here:

1. EventBridge Rule -- triggers the Search lambda every minute
2. DynamoDB Table -- Contains the alert records
3. Search Lambda -- Pulls alerts from DynamoDB and polls BMS
4. SQS Queue -- When a movie is found, the results are pushed to this queue
5. Notification Lambda - Parses result from the queue and sends notification

EventBrige works like a Cron job. I went with DynamoDB for storing the records as it's pretty simple to get started with and I have room to optimize how I query this table in the future.

I could've used a single lambda function to poll and send the notification, but I wanted to decouple these two jobs as it allows me to write more independent functions to send notifications to a variety of interfaces in the future. I can have a function for sending alerts via email, pushbullet, SMS or whatever I wish, each function would work with the same result format.

## Putting it up and online

I am partial to AWS SAM when it comes to working with serverless on AWS, it's very simple to define my resources and have them be deployed with a single command.

Once the stack is up, I just need to create entries in the DDB table with my movie specifications. I was short on time, and as I wrote this thing, I knew exactly what and how I needed to put the data in. I simply used the AWS Management Console to add the entries.

![Alerts in DynamoDB Table](images/diomedes_search_alerts.jpg "Alerts in DynamoDB Table")

We need following pieces of crucial information:

1. Get `RegionName` and `RegionCode`
2. Get `TheaterCode`
3. ChatID of the Telegram Chat.

With BookMyShow, since we are only interested in when tickets go sale on theaters of our choice, we need to know the Theater Code assigned by BMS, and theaters are zoned by regions (and sub-regions but it's not important for us)

I already know these details for my theaters, but I'll be writing a small CLI utility in goBMS that allows you to list all Regions with their region code, and then narrow down theaters in that region to get these 3 pieces of information.

Since I'm interested in receiving alerts in our group chat on Telegram, I created a Telegram Bot and added it to my group chat, and noted down the ChatID for our group.

With these in hand, we can fill the rest up pretty easily.

1. Date - [in the one true format](https://twitter.com/ben11kehoe/status/1415478139996704774)
2. Format - the dimension we talked about earlier
3. Completed - Leave this at False, we use this for querying alerts.
4. Language
5. MovieName - Since we do a string matching on the BMS API Response to check if a movie is available, we need this to match exactly how BMS has it.

## And away we go

With everything in place, we just had to sit and wait for sales to commence. From previous experience, bookings on BMS open a week before release. We finally got the notification on December 12th, Sunday evening.

![Alert on Telegram](/images/diomedes_search_tg_notification.jpg "Alert on Telegram")

I could've formatted the message better to highlight the date and format but it's not bad for a weekend's work. We got the best seats in our theater. The movie lives up to the hype then some, so this was all worth it! Make sure to catch it in IMAX if possible.

The movie ended up setting several box office records, and ended up [crashing Fandango and AMC Theater websites](https://www.cnet.com/news/spider-man-no-way-home-tickets-are-being-scalped-on-ebay-for-ridiculous-prices/) during pre-sales. So I'm glad I didn't depend on manually checking for the tickets.

## What's in the future?

There's a lot I can do to polish this. The first thing is obviously the formatting of messages.

But what I'm more interested in are performance optimizations, even though no one except me will ever use this unless I build a usable frontend for this. Either way, I'd like to think they'll be good excuses to learn stuff. Premature optimizations are fair game here.

### Concurrent jobs

This is something I never did in the Django App, and I quickly noticed how bad it made things. I remember it took 9 minutes to process 200 jobs, and it could've been quicker if I made each job run concurrently. Python has plenty of options, from [asyncio](https://docs.python.org/3/library/asyncio.html) to [concurrent.futures](https://docs.python.org/3/library/concurrent.futures.html) that make this possible. I have used them in the [past](https://kanishk.io/posts/telegram-push-notification-service/) as well.

With Go, this gives me a nice excuse to play around with goroutines.

### Querying DynamoDB

If you read my code right now, you'll see that I'm making a rookie mistake, i.e. not modelling my table properly. What I'm currently doing is pulling all my records and filtering them to select records that aren't marked completed.

Since I was in a hurry I went with the most basic approach. This gets more expensive to run as the number of records grow, I'll always be pulling all the records and sifting through them to select what I want.

To optimize this, I'll need to make use of sparse GSIs offering by DynamoDB. It'll allow to only pull records which have a property set. So I'll simply keep an attribute "incomplete" for all new alerts, and delete the attribute when I've sent an alert for it.

### Minimizing API calls

The less number of calls I need to make to BMS, the faster jobs can finish. This involves caching data from BMS responses so I don't have to query for it in subsequent runs. This has been on my list for a long time actually, and something I originally intended to work on in the Django app.

I have some ideas on which pieces of data I can cache to lower my turn around time for each job. I'll be exploring those but only if I can get the above ones done first. This is a low hanging but highly valuable fruit.

In case you want to check out the code, it's available at [ArionMiles/diomedes-search](https://github.com/ArionMiles/diomedes-search)

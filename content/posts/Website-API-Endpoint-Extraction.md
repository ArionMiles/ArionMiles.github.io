---
title: Webscraping with APIs - A guide to find APIs for websites
date: 2018-10-11
type: "post"
description: "Find APIs for any website or service"
tags: 
 - webscraping
---

I've recently been involved in a couple of projects where I had to extract information from a website. Obviously, the first thing
you'd think of is using BeautifulSoup or Scrapy to parse the HTML and extract our data. But this method can get really messy
really fast, when you have to extract hundreds of different items periodically. This entails finding hundreds of different CSS and XPath information from the website. Oh, and then one day your script just doesn't work, they've changed their markup. Cue investing hours of your time in the same mundane task once again.

But there's a way to reduce this work and almost ensure that your code never requires reworking again (at least the webscraping
parts of it).

![Morpheus](/images/api-endpoint-extraction-morpheus.jpg "Morpheus")

Before learning that, we need to familiarize ourselves with the different kinds of rendering techniques used in Web Development. 

## Server-Side Rendering
Here, your browser makes a request to the server, and the server sends the entire webpage, pre-built, to your browser. This
method was used a lot until websites grew into web applications, which required a lot of dynamic elements, which meant sending
a request for small changes and receiving the entire document was very costly. For small changes, the server had to re-generate the entire page once again, not to mention the waste of bandwidth it caused.
Enter Client-Side Rendering.

## Client-Side Rendering
Client side rendering is a technique where the browser sends a request for your website's page and initially gets the entire webpage pre-built, but once you start interacting with it, it receives small chunks of data in the background and updates the original document to reflect the changes. You might have noticed it on YouTube where selecting a video from the sidebar doesn't reload the entire page, only portions of it.

The way it works is that each time you interact with a part of the website, it triggers an AJAX (Asynchronous Javascript and XML) request which receives a response in either XML or JSON (very popular these days) and it's parsed by the client-side javascript code of the website to properly display each value of the response wherever necessary.

This method is very performance efficient, as it doesn't put unnecessary load on the webserver or consume more bandwidth, allowing you to serve more users at the same time.

This method is also the reason why we will save on hours of wearisome webscraping.

## Chrome Dev Tools

Inspect Element. One of the first things I discovered one day in my browser and it just sort of ticked off something in me which led me to take up programming and tech in general.

For monitoring all the incoming and outgoing requests for your website, simply go to the Dev Tools (F12) and navigate to the Network tab. We're looking for AJAX calls mainly, so use the XHR (XMLHttpRequest) filter. Now, it's simply a matter of inspecting the entries and looking at the response to find the correct API endpoint. If the tab has no URLs after you've filtered with XHR request, simply refresh the page, or interact with a part of the website the data for which we want an API for.

## Postman
[Postman](https://www.getpostman.com/apps) is a great GUI tool for inspecting API responses and modifying-tweaking it before implementing it in your code. In fact, Postman provides automatic code generation for sending requests for number of programming languages and multiple libraries (python - requests/urllib, go, js, etc).

Paste the API endpoint and play around with the response. You can save multiple endpoints into a single collection which can be used by your friends or collaborators too.

## This endpoint is sending HTML WHAAT!? - Enter Wireshark

![Sesame Street](/images/api-endpoint-extraction-sesamestreet.jpg "Sesame Street")


Yeah, some sites are like that. Some people just wanna watch you scrape and burn. They communicate using AJAX requests on the frontend, but the responses from the AJAX requests is completely pre-rendered html code.

This part made me think how these websites work with their mobile apps? They must be using the same endpoints for website and app, yeah? it's easier that way, isn't it? Turns out I'm very wrong. They have separate API endpoints for their mobile Apps. 

How do you access these endpoints? There are several ways to approach this.

1. Connect your phone to your internal Wi-Fi access point (before connecting, check advanced options and set a manual proxy).
    Note: This method does not work for all network data. Some connections ignore this setting.
2. **For Rooted Devices:** Install 'Shark for Root' application on your device. It will capture ALL traffic. It will generate dump files that can be analyzed on your PC using [Wireshark](https://www.wireshark.org/#download).
3. The best way: Setup your PC as a Wi-Fi access point and make your Android/iOS device to use this Wi-Fi connection, then sniff the traffic using the same Wireshark application.

[Source](https://stackoverflow.com/a/21757608)

So the third option suits me best, I don't have to bother with rooting or screwing with proxies. I just need Wireshark. On Windows 10 you can easily start a Mobile Hotspot from the Notifications area or just do the following:

`Settings > Network & Internet > Mobile Hotspot` and turn it on.
For older versions, you can checkout [Connectify](http://www.connectify.me/) which makes hotspot creation easier.

For Ubuntu and other Debian based distros:
Check out this [AskUbuntu](https://askubuntu.com/a/609199) answer and [UbuntuHandbook blogpost](http://ubuntuhandbook.org/index.php/2014/09/3-ways-create-wifi-hotspot-ubuntu/).


![Wireshark](/images/api-endpoint-extraction-wireshark.png "Wireshark")


### How it works:
Your computer is acting a hotspot which your phone connects to. And your computer is connected to the Internet via Wi-Fi or LAN, and so all the traffic, each Web request you make on your phone goes through your computer first, and then to internet.

Since Wireshark resides on your computer, it is able to capture all that traffic (along with your computer's traffic as well, but we're not concerned with that) and is able to search through it.

As you begin sniffing, interact with the mobile app in some way for it to trigger talking to the remote servers for data. After capturing packets is over, apply `http` filter, to look for HTTP requests in particular, now simply go to Find (Ctrl + F), set a `String` Display Filter and type out the name of the website/app, and surely enough, you will be able to find that the app is making GET/POST request with certain parameters/data. 

Go to the `Hypertext Transfer Protocol` section in the middle window of Wireshark, and scroll down till you see a `Full Request URI` field, it will look like a link. `Right click > Copy > Value` to copy the URL and when you put that URL in Postman, you'll see a JSON response (most likely) which can be easily parsed by your code. Keep tweaking the request in Postman to find the best suited configuration and when you're ready, use it in your code.


## Ethics
We're basically abusing an undocumented internal API here, and the site owners most probably will never appreciate this behaviour. So good luck using this technique for a commercial product. However, if you're doing it for learning purposes, keep in mind to never push their data to a repository somewhere, or sell that dataset. As long as you don't bombard them with tremendous amounts of requests I think you'll be fine.

## Conclusion
Using XHR endpoints to receive JSON data from the website makes it easier to parse the data using built-in JSON/XML parsers in any high level language of your choice. Saves you a bunch of time you'd waste on figuring out CSS selectors and XPath. If a site changes their UI, your code still remains functional. You only have to touch the code if they make changes in their API, which is very rare.

If you have any questions or suggestions, feel free to reach out to me on Twitter!
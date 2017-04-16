---
layout: post
title:  Attacking Microsoft Edge to identify users by leaking URLs from Fetch requests
---

## TL;DR
I found that Microsoft Edge exposes the URL of any Fetch response, in contradiction to the specification. I prove this is a problem by showing it is possible to identify users by crafting a Fetch request to a URL that will redirect to a URL with the user's username (e.g. https://facebook.com/me to https://facebook.com/username). This may in some cases obsolete identification by IP or browser fingerprinting.

## Introduction

Although I am generally less interested in web development issues, I've recently had the time to play with some of the relatively new web APIs, such as WebSockets and [Service Workers](https://developers.google.com/web/fundamentals/getting-started/primers/service-workers). I tried testing the boundaries of Service Workers, hoping to find anything that compromises the users' privacy. I didn't find anything with Google Chrome, but the first few minutes playing with Microsoft Edge I found the following information exposure in the Fetch API. 

I would generally not bother writing about this and just let Microsoft resolve it, but their security team refused to acknowledge it as an issue, so I hope that publishing it may convince them otherwise.

Please make yourself familiar with [Fetch](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API/Using_Fetch). In short, it is the cleaner, modern replacement for XMLHttpRequest (XHR).

## The Issue

By default, requests made with fetch respect [CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/Access_control_CORS) and deny access to inappropriate resources. It is possible however to make a request with "no-cors" mode, enabling fetch to bypass CORS. The response of such a request should be "opaque", meaning it's content should be empty (so that an attacker could not gain access to cross-origin requests made with the user's session). [The specification](https://fetch.spec.whatwg.org/) states the following:

> An opaque filtered response is a filtered response whose type is "opaque", url list is the empty list, status is 0, status message is the empty byte sequence, header list is empty, body is null, and trailer is empty.

The vulnerability is that in Microsoft Edge the url property of an opaque response is not an empty string, but it is the full URL of the response returned after following redirects.

This may not sound like an issue at first since no confidential information should be stored in the URL. That's correct. However, many websites use redirection to redirect the user to a personal page. For instance, a website redirecting https://example.com/profile to https://example.com/my_username. 

Some real life examples used in the PoC are https://facebook.com/me, https://youtube.com/profile and https://docs.com/me. Redirection to the user page is just  an example and there are other possibilities to exploit this issue, left to the reader as an exercise.

## Proof of Concept

[Test your browser](/poc-fetch) or [see the GitHub project](https://github.com/zelivans/poc-fetch) (run poc_facebook.py for the Facebook example)

I originally tried to find a Microsoft service that the issue applies to, so the first PoC targets [https://docs.com](https://docs.com). I later modified it a little to target Facebook. The simpler version only proves the URL leak happens with Docs, Facebook and Youtube. Anyhow, the effect is that without any user interaction or consent the user's identity is leaked to the attacker's website.

I've tested the PoC on Microsoft Edge 40.15063.0.0 (slow ring) and on Microsoft Edge 38.14393.0.0 (current release).

## Microsoft's response

+ 27.3.2017 - Sent report and PoC to MSRC (Microsoft Security Response Center)
+ 27.3.2017 - MSRC opened a case for the issue
+ 7.4.2017 - MSRC have determined this is not a security issue

> Hello Ariel,
>
> We have determined that this does not met the bar for security servicing through a security bulletin as the only information disclosed is the URL, which should not contain sensitive data. The product team has opened a tracking item to address this in a future version update.
>
> We appreciate your report. If you have any additional concerns or questions, please let me know.
>
> Cheers,
>
> \-

+ 13.4.2017 - Following some other emails I made sure it is OK by MSRC to publish this.

Additionally, I don't know if related or not, on 1.4.2017, Microsoft sent the following email:
![docs.com email](/assets/docs.com.png)
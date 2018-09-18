---
layout: post
hidden: true
title: Attacking Microsoft Edge to identify users by leaking URLs from Fetch requests
---

## TL;DR
I found that Microsoft Edge exposes the URL of any JavaScript Fetch response, in contradiction to the [specification](https://fetch.spec.whatwg.org/). This is a problem because it is possible to identify users by crafting a Fetch request in a webpage that will redirect to a URL containing the visitor's username (e.g. requesting https://facebook.com/me will result in https://facebook.com/username). This may obsolete identification by IP or browser fingerprinting.

## Introduction

Although I am generally less interested in web development issues, I've recently had the time to play with some of the relatively new web APIs, such as WebSockets and [Service Workers](https://developers.google.com/web/fundamentals/getting-started/primers/service-workers). I tried testing the boundaries of Service Workers, hoping to find anything that compromises the users' privacy. I didn't find anything with Google Chrome, but the first few minutes playing with Microsoft Edge I found the following information exposure in the Fetch API. 

I would generally not bother writing about this and just let Microsoft resolve it, but their security team refused to acknowledge it as an issue, so I hope that publishing it may convince them otherwise.

Please make yourself familiar with [Fetch](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API/Using_Fetch). In short, it is the cleaner, modern replacement for XMLHttpRequest (XHR).

## The Issue

By default, requests made with fetch should respect [CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/Access_control_CORS) and deny access to inappropriate resources. However, it is possible to make a request with "no-cors" mode, enabling fetch to bypass CORS. The response of such a request should be "opaque", meaning it's content should be empty (so that an attacker could not gain access to cross-origin requests made with the user's session). [The specification](https://fetch.spec.whatwg.org/) states the following:

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
+ 13.4.2017 - Following some other emails I made sure it is OK by MSRC to publish this
+ 6.7.2017 - MSRC had assigned CVE-2017-8504 for this issue and a patch was released in the 6/13 Patch Tuesday

**Update (24.4.2017):** I've reached MSRC once again after this post became popular [on Reddit](https://www.reddit.com/r/netsec/comments/65ucsn/attacking_microsoft_edge_to_identify_users_by/) and social media. Their response clarified that they do treat this as a security issue and assured a patch is to come soon.

**Final update:** Jonathan from MSRC had assigned a CVE for this issue attributed to me in their [acknowledgement page](https://portal.msrc.microsoft.com/en-us/security-guidance/acknowledgments).
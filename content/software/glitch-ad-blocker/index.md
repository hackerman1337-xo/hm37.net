---
title: "Glitch Ad Blocker"
date: 2020-12-09T15:00:00-00:00
tags: [twitch]
---

[Twitch](https://twitch.tv) has rolled out a new update for injecting ads in streams. This time, they're injecting ads directly into the stream feed using a product they've dubbed (SureStream)[https://twitchadvertising.tv/ad-products/surestream/].

## Intro

I won't dive into the technical details of how it works, but a high-level explanation is this: Twitch serves broadcasts over HLS, which means your browser/app/console/etc will download small chunks of the live broadcast and piece them together. Instead of buffering a 2 minute segment of video, for instance, your browser will download 60 2-second clips and piece them together. Twitch uses some clever cookie manipulation to identify a user and their entitlement to ads to change what those 2-second clips show. In this way, you will often be served 30, 60, or 90 seconds of ads **by Twitch changing the stream chunks from the broadcaster to the ad video**.

Thankfully, there's a great community of nerds who will always work to find ways around these intrustive policies. Today, my Glitch Ad Blocker extension is available on all major browsers, and offers client-side blocking of ads on Twitch. Currently, mobile is unsupported (as the method used for bypassing ads isn't something you can do safely on mobile, and most mobile browsers don't support add-ons), but for desktop, this method works great.

## Links

If you'd like to install the ad blocker, here are the links for the major browsers:

- Mozilla Firefox: [Glitch Ad Blocker on Firefox Add-Ons](https://addons.mozilla.org/en-US/firefox/addon/glitch-ad-blocker/)
- Google Chrome/Brave: [Glitch Ad Blocker on Chrome Web Store](https://chrome.google.com/webstore/detail/glitch-ad-blocker/lipdfgjnoaojanblcgnjfiiognpihldc)
- Microsoft Edge (Chromium): [Glitch Ad Blocker on Edge Add-Ons](https://microsoftedge.microsoft.com/addons/detail/glitch-ad-blocker/ebbdhbccddkbnpoffagbekcjohamjhgd)

## Source Code

The source code for this add-on is available on my GitHub repository here: [https://github.com/hackerman1337-xo/glitch-twitch-adblock](https://github.com/hackerman1337-xo/glitch-twitch-adblock). Instructions on building the add-on are there as well, if you'd prefer to load the add-on or make your own changes first.
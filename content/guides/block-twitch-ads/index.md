---
title: "Blocking Twitch Ads (Oct 2020 Update)"
date: 2020-10-01T19:00:00-00:00
tags: [twitch]
---

Twitch recently rolled out a new update to introduce mid-roll ads on the platform. This broke many ad blockers, including my favorite -- uBlock Origin. Below are instructions on fixing uBlock.

If you'd like to install uBlock Origin, here are the links for the major browsers:

- [Firefox Add-ons](https://addons.mozilla.org/en-US/firefox/addon/ublock-origin/)
- [Chrome Store](https://chrome.google.com/webstore/detail/ublock-origin/cjpalhdlnbpafiamejdnhcphjbkeiagm?hl=en) (Also works on Brave Browser)
- [Edge Add-ons](https://microsoftedge.microsoft.com/addons/detail/ublock-origin/odfafepnkmbhccpbejgmiehpchacaeak) (for Chromium Edge)
- [Microsoft Store](https://www.microsoft.com/en-us/p/ublock-origin/9nblggh444l4) (for Edge Legacy)

{{< video src="ublock.mp4" muted="true" autoplay="true" loop="true" controls="no" >}}

## uBlock Origin Steps

1. Open your browser, and navigate to uBlock Origin settings
- On Firefox, click the uBlock Add-on icon, then click the settings icon
2. Check the option "I am an advanced user", and then click the settings cog next to "I am an advanced user"
3. At the bottom of the advanced settings, you'll see a line that says `userResourcesLocation unset`. Replace `unset` with `https://gist.github.com/hackerman1337-xo/524e2c4e655e3f1014eac01d104eb832`, and click "Apply changes". Note: you can view the gist content here: [https://gist.github.com/hackerman1337-xo/524e2c4e655e3f1014eac01d104eb832](https://gist.github.com/hackerman1337-xo/524e2c4e655e3f1014eac01d104eb832))

That's it! Enjoy ad-free Twitch without giving them any of your money.
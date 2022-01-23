---
layout:   post
title:    Getting Honeybadger Uptime Alerts via SMS with Zapier
date:       2019-09-04 09:00:00
categories: honeybadger, zapier, webhooks, sms
---

*UPDATE: Shortly after posting this Honeybadger was [kind enough to let me know](https://twitter.com/honeybadgerapp/status/1169684671556308992)
that [this is functionality that they offer directly](https://docs.honeybadger.io/guides/profile.html#configuring-personal-alerts)* 😁

After using [Honeybadger](https://www.honeybadger.io/) Uptime Monitoring for a
few months, I found that these updates were sometimes getting lost in the
everyday chatter of my [error messages that I have posted to a Basecamp chat](/posts/2019/07/honeybadger-to-basecamp)
via Zapier.

With that in mind I came up with the idea to create a new
[Honeybadger webhook alert](https://docs.honeybadger.io/guides/services.html#1-select-the-webhook-integration)
that would post only uptime updates to Zapier.

Below is an overview of the Zap I came up with:

<div class="text-center">
<img src="/images/2019/09/zapier-text-alert.png" style="width: 30%;" class="m-auto shadow-md">
</div>

With that now in place I now have Zapier text me whenever something big happens... which hopefully is rarely to never :)

---
layout:   post
title:    Piping Your Honeybadger Events to Basecamp with Zapier
# link:     https://gist.github.com/kylekeesling/b7d2571de44f2ad8f5cf
# source:   #used for naming the source of your link
date:       2019-07-03 13:30:00
categories: basecamp, honeybadger, zapier, webhooks
---

My team has been working on consolidating and improving our communication tools
and practices, and as part of that we've decided to ween ourselves off of
[Slack](http://slack.com) and totally leverage [Basecamp](http://basecamp.com)
for all internal and customer communications.

One of the major things that the team appriciated in Slack was the #dev channel I had created that
piped all of our deploys, pull requests, and errors into one central location,
allowing everyone to know what's going on without having to tap someone on the shoulder.

Basecamp provides an easy way, using a [chatbot](https://github.com/basecamp/bc3-api/blob/master/sections/chatbots.md),
to feed Github events into a Campfire chat, but getting our application errors in
wasn't quite as straightforward.

We use [Honeybadger](https://www.honeybadger.io/) to track our errors, and while
they [offer many integrations](https://docs.honeybadger.io/guides/integrations.html),
Basecamp isn't one of them — enter [Zapier](https://zapier.com).


## How'd I do it?
Zapier made it a very simple process:

1. create a [catch webhook](https://zapier.com/apps/webhook/integrations) to catch [incoming webhooks from Honeybadger](https://docs.honeybadger.io/guides/services.html#1-select-the-webhook-integration)
2. create a POST webhook action that massages the JSON payload from Honeybadger, into [the format Basecamp expects for a chatbot](https://github.com/basecamp/bc3-api/blob/master/sections/chatbots.md#create-a-line)

<br><br>

### The Result in Basecamp

![Honeybadger Chatbot Example](/images/2019/07/honeybadger-chatbot1.png)

Using the `details` and `summary` elements gives you the ability in Basecamp to
create a collapsable post. In the sample above notice the little black right arrow,
if you click it you can even get a little sample of the stacktrace. Here's the value
of the `contents` key I used for the chatbot payload:

<div class="overflow-x-scroll">
  {% highlight html %}
    <details>
      <summary>
        <strong>💥({xxx__fault__id}){xxx__message}</strong><br>
        <a href="{xxx__fault__url}">View in Honeybadger</a>
      </summary>
      <br><hr><br>
      <pre>{xxx__notice__application_trace}</pre>
    </details>
  {% endhighlight %}
</div>

And here's a preview of what the expanded preview looks like:

![Honeybadger Chatbot Example w/ Stacktrace](/images/2019/07/honeybadger-chatbot2.png)

## Summary
We are still in the process of fulling moving over to Basecamp, but so far I've not missed
Slack nearly as much as I thought I would.

In all honestly the things we've missed
the most from Slack are the GIPHY integration, and the quick access to screen collaboration
tools, so if you have any suggestions on how your team addresses those areas, [I'd
love to hear about it](https://twitter.com/kylekeesling).

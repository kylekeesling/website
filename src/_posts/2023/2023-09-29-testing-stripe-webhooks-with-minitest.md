---
layout: post
title:  "Testing Stripe Webhooks with Minitest"
tags: rails, ruby, stripe
date:   2023-09-29 12:00:00 -0400
categories:
  - rails
  - testing
  - stripe
blurb: |
  An easy way to test your Stripe webhooks Minitest.
---

As I was looking to sure up some billing code in one of my projects, I figured it was finally time that
I created some tests for the various [Stripe webhooks](https://stripe.com/docs/webhooks) that we used.

I was a bit dishearted that the only webhook testing examples I could find when pertained to
RSpec, so with that I rolled up my sleaves and dug into the internals of both the `stripe_event` gem
as well as [Stripe's offical gem](https://github.com/stripe/stripe-ruby).

After multiple rounds of trial and error I was able to discern how Stripe generates the signature
for webhooks, and how I could mimic it to test my endpoint. You can see that below in the
`webhook_header` method.

The only other bit of prework you'll need to do is to grab sample payloads for each of the webhook
events you'd like to test and save them as a JSON file in `test/fixtures/stripe/#{event}.json`,
as can be seen illustrated in the `webhook_payload` method. If you aren't sure where to grab sample
payloads from, the [Stripe CLI](https://stripe.com/docs/stripe-cli) has a `trigger` command that
can make gathering them much easier.

The only other varaible you may have is the path your your webhook endpoint, which in my
case is `stripe_event_path`, as seen in the `deliver_payload` method.

Take all of that and you get the following test case can be used as a base class for any
tests you may need to write:

```rb
# test/stripe_test_case.rb
require "test_helper"

class StripeWebhookTestCase < ActionDispatch::IntegrationTest
  SIGNING_SECRET = ENV["STRIPE_SIGNING_SECRET"]

  def deliver_payload(event:)
    params, headers = webhook_setup(event)

    post stripe_event_path, params:, headers:
  end

  def webhook_setup(event)
    payload = webhook_payload(event)
    [payload, webhook_header(payload:)]
  end

  def webhook_payload(event)
    File.read(Rails.root.join("test", "fixtures", "stripe", "#{event}.json"))
  end

  def webhook_header(payload:)
    time = Time.now
    signature = Stripe::Webhook::Signature.compute_signature(time, payload, SIGNING_SECRET)
    header = Stripe::Webhook::Signature.generate_header(time, signature)

    {"Stripe-Signature": header}
  end
end
```

Once you have this test case, here's an example test that could be used:

```rb
# test/webhooks/stripe/general_webhook_test.rb
require "stripe_webhook_test_case"

class Stripe::GeneralWebhookTest < StripeWebhookTestCase
  test "webhook endpoint returns proper status code" do
    deliver_payload(event: "charge.succeeded")

    assert_response :ok
  end

  test "webhook endpoint returns proper status code when no headers are provided" do
    post stripe_event_path, params: webhook_payload("charge.succeeded")

    assert_response :bad_request
  end

  test "webhook endpoint returns proper status code when improper headers are provided" do
    post stripe_event_path, params: webhook_payload("charge.succeeded"),
      headers: {"Stripe-Signature": "#{Time.now.to_i},v1=badsignature"}

    assert_response :bad_request
  end
end
```

In this project I also happen to be using the [`stripe_event` gem](https://github.com/integrallis/stripe_event),
so a good deal of the actual route/endpoint logic is abstracted from my app. I'm currently considering rolling
my own completely and removing this gem as a dependancy, as I've done in another project already, but
that's a topic for another post. The Stripe docs also have
[a really good example](https://stripe.com/docs/webhooks#example-endpoint) one could use as a starting point.

---
layout:   post
title:    Setting Up Proper Amazon S3 Permissions for ActiveStorage
date:       2020-01-28 17:43:00
categories: rails, ruby, activestorage, s3
---

If you've found yourself marveling at how cryptic and impenetrable understanding AWS services, then you are definitely not alone. For much too long have I relied on doing a quick web search and blindly copying and pasting settings and policies, so when I found myself doing it again this week for [one of my applications](https://passtesting.com/tools/service-providers) I decided it was time to slow down and actually understand what it was that I was doing.

## The Problem

I'm creating a demo environment that needs to be completely separate from our production data, so part of that entails creating its own S3 bucket for use with [ActiveStorage](https://edgeguides.rubyonrails.org/active_storage_overview.html). Having not really done this since ActiveStorage was first released I begin DuckDuckGoing (is this even a thing?) for a how-to.

Some perfectly fine write-ups and tutorials popped up on the first page, but I noticed that all of them recommended setting up an [IAM User](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users.html) with `AmazonS3FullAccess` permissions. As soon as I looked at the description for this policy it threw up a _big_ red flag:

> `AmazonS3FullAccess: Provides full access to all buckets via the AWS Management Console`

Full access to all buckets? That's gonna be a big nope from me.

## The Solution

At an absolute minimum the permissions need to be locked down to a single bucket, but ideally we'd only give the app the ability to perform solely the actions that are required for the framework to function properly. Luckily there's [a callout right in the Rails guide](https://edgeguides.rubyonrails.org/active_storage_overview.html#amazon-s3-service) that states:

> The core features of Active Storage require the following permissions: `s3:ListBucket`, `s3:PutObject`, `s3:GetObject`, and `s3:DeleteObject`. If you have additional upload options configured such as setting ACLs then additional permissions may be required.

Assuming you already have a bucket created, we just need to [create an IAM User](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_create.html) so we can generate the `access_key_id` and `secret_access_key` necessary to configure the `config/storage.yml` Amazon service.

Once you have your IAM user created you can use the policy template I've built down below as a guide to either create and attach the policy to a group you add your IAM user to, or just attach the policy directly to the user, which is what I did.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "VisualEditor0",
      "Effect": "Allow",
      "Action": [
          "s3:PutObject",
          "s3:GetObject",
          "s3:DeleteObject"
      ],
      "Resource": [
          "arn:aws:s3:::BUCKET_NAME_GOES_HERE/*"
      ]
    },
    {
      "Sid": "VisualEditor1",
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::BUCKET_NAME_GOES_HERE"
    }
  ]
}
```

## Final Notes
To their credit, one or two articles said something to the extent of "you may want to do this differently in production", but let's be honest, most people don't read the entire article and/or will not remember to do this, so I find it to be a much better idea to set things up right from the start.

Hopefully this helps you in your ActiveStorage-related endeavors, and if you have any suggestions or comments on how you handle S3 permissions, I'd love to hear them!

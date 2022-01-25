---
layout:   post
title:    Migrating Your Assets from Paperclip to ActiveStorage
# link:     https://gist.github.com/kylekeesling/b7d2571de44f2ad8f5cf
# source:   #used for naming the source of your link
date:       2018-04-26 08:00:00
categories:
  - ruby
  - rails
  - activestorage
  - paperclip
---

With the release of Rails 5.2 there is now a native, built in way to handle asset uploads and management called [ActiveStorage][activestorage], making the need to use gems like [Paperclip][paperclip], [Carrierwave][carrierwave], or [Fog][fog], with Paperclip going so far as writing a [migration guide][paperclip-migrate-issue]; implying that it may not be long for this world.

To be honest I didn't even know that Paperclip had written a [migration guide][paperclip-migrating] until I started writing this post, and while it's fairly comprehensive, it doesn't do what in my mind is one of the most important tasks - migrating the files on the remote host.

With that in mind I've used the following rake tasks with great success to move assets for many of my models.

There are two flavors here, migrating assets while changing their name, or just plain old moving.

## Moving an Asset While Changing Its Name

In my case I had a `User` class with a Paperclip attachment called `headshot`. I never really liked that we used the term headshot, so this was the perfect opportunity to change it to `avatar`.

{% gist 87231b253ab15b0ea99f65f755ebd576 user.rb %}

This scenario is surprisingly easier due and the fact that we don't have to worry about a naming collision (`headshot` to `avatar`), which means we can still use paperclip's built in URL helpers in our rake task.

{% gist 87231b253ab15b0ea99f65f755ebd576 users.rake %}

Just remember to leave your Paperclip declarations on your model until after you run your rake task.

## Moving an Asset As-is
In reality, you'll likely want to keep the same name, which requires us to get a little more crafty. In my case I have an `Organization` class with a `logo`. We wanted to keep using that term, but in order to do so, we have to remove the Paperclip declarations from the model, otherwise would couldn't refer to all the new ActiveStorage goodness.

{% gist 87231b253ab15b0ea99f65f755ebd576 organization.rb %}

To get around this we just need to utilize the actual asset URL in our rake task.

{% gist 87231b253ab15b0ea99f65f755ebd576 organizations.rake %}

This rake task is pretty straightforward, but what got a little sticky was dealing with filename extensions, mainly the fact that sometimes uploaded files don't have them, so that's what lines 5 & 6 are about.

## A Few Considerations

There are a few tradeoffs that you'll have to consider if you use this method though.

The first advantage that you get is that these tasks can be ran from any server, including your dev environment, even before you deploy any actual code.

In my case I ran the rake tasks on my laptop, and as soon as the updated code was deployed to the server, the assets were already there. The only thing left to do is to remove the old assets when you're ready, but they are still there in the event you need to rollback your deploy.

The biggest downside to this method is simply the fact that you are duplicating all of your assets temporarily. In my app I was able to move thousands of image files in a matter of hours, but if you have millions of records, or very large assets, this method might not be your best option.

[activestorage]: http://edgeguides.rubyonrails.org/active_storage_overview.html

[paperclip]: https://github.com/thoughtbot/paperclip/
[paperclip-migrate-issue]: https://github.com/thoughtbot/paperclip/pull/2568
[paperclip-migrating]: https://github.com/thoughtbot/paperclip/blob/master/MIGRATING.md

[fog]: https://github.com/fog/fog

[carrierwave]: https://github.com/carrierwaveuploader/carrierwave

---
layout: post
title: Migrating from Sidekiq to Solid Queue
tags: rails, ruby, solid_queue, sidekiq
date:   2024-01-03 12:00:00 -0400
categories:
  - rails
blurb: |
  A simple guide for migrating a vanilla Sidekiq implementation to Solid Queue
---

With the recent release of [Solid Queue](https://github.com/basecamp/solid_queue), and a little bit
of extra free time due to the holidays, I made a decision to migrate my apps away from
[Sidekiq](https://github.com/sidekiq/sidekiq). The idea of simplifying my infrastructure requirements
by removing Redis from the equation, and the relatively vanilla requirements I have for a background
queuing system were just enough to convince me this was a worthwhile exercise.

I won't bore you with every machinations of copying, pasting, and deleting code since it's realtively
straightforward, but I did think it was worth sharing a few highlights, considerations, and
gotchas I came across.

## The Migration

The switchover in my case was very straightforward since my use of Sidekiq was all via
[Active Job](https://guides.rubyonrails.org/active_job_basics.html). It consisted of:

- Swapping out the gems in my `Gemfile`
- Running [the Solid Queue installation scripts](https://github.com/basecamp/solid_queue#installation-and-usage)
- Removing the Sidekiq UI mounting lines from `routes.rb`
- Deleting `sidekiq.rb` from `config/initializers`
- Tweaking my [Solid Queue config](https://github.com/basecamp/solid_queue#configuration)

## Configuration
I migrated two apps to Solid Queue. One of them mainly uses ActiveJob to send emails and run reports,
so I simply replied on the default config, but my other app does utilize multiple queues, so I
ended up with the following config for it:

```yml
# config/solid_queue.yml
default: &default
  workers:
    - queues: "*"
      polling_interval: 2
    - queues: real_time
      polling_interval: 0.1

development:
  <<: *default

test:
  <<: *default

production:
  <<: *default
```

My goal here was to create a priorty queue, called `real_time`, so I decided to add a second worker
dedicated soley to those jobs, and to poll for them at a much higher frequency.

## A Few Considerations
My app with the completely vanilla config deployed and worked without a hitch, but I did run into a
few hiccups with my custom config.

### DB Connection Pool
I deploy my apps on Heroku, and by default the app is setup to look at the `DB_POOL` environmental
variable to set the connection pool in `config/database.yml`. This is fine for app instances but
since I'm running a dedicated worker dyno for Solid Queue, some extra work was needed.

By default each Solid Queue worker has 5 threads, meaning that since my app has two workers, I
needed a connection pool that allowed 10 connections. I was able to accomplish this by using a specfic
enviroment variable, `SOLID_QUEUE_DB_POOL`, as is suggested [in the Heroku docs](https://devcenter.heroku.com/articles/concurrency-and-database-connections#background-workers).

```yml
# Procfile
web: bundle exec puma -C config/puma.rb
worker: DB_POOL=$SOLID_QUEUE_DB_POOL bundle exec rake solid_queue:start
release: bundle exec rails db:migrate
```

### Error Handling

_**Note**: A previous version of this post referred to the `on_thread_error` setting to catch job errors. [After some more
digging](https://github.com/basecamp/solid_queue/issues/120#issuecomment-1894413948) I realized this
setting is not for job errors but rather actual errors that occur that are related to the thread directly._

By default the library silently handles errors and marks them as failed, but I'm prefer being proactively
notified when something happens. To do this you'll need to leverage an `around_perform` callback to wrap
your jobs in an error handler, then report them as necessary

```ruby
 class ApplicationJob < ActiveJob::Base
  around_perform do |job, block|
    capture_and_record_errors(job, block)
  end

  def capture_and_record_errors(job, block)
    block.call
  # I had to use rescue here instead of a `Rails.error` block because Honeybadger ignores the `Rails.error.report` call
  # in favor of their own error handler, which is fine in most cases, but unfortunately doesn't work here. Report would be
  # great here because it re-raises the error, but instead I have to do that manually
  rescue Exception => e
    Rails.error.set_context(**error_context(job))
    Rails.error.report(e)
    raise e
  end

  def error_context(job)
    {
      active_job: job.class.name,
      arguments: job.arguments,
      scheduled_at: job.scheduled_at,
      job_id: job.job_id
    }
  end
end
```

A huge thanks goes out to [Rosa Gutierrez](https://github.com/rosa) for not only her wonderful work on this gem, but also for
taking the time to respond to Github issues.

### Failures and Retries
By default, [Sidekiq has a built-in retry mechanism](https://github.com/sidekiq/sidekiq/wiki/Error-Handling),
but [this is not the case for Solid Queue](https://github.com/basecamp/solid_queue#failed-jobs-and-retries).
If automatic retrying is important for you you'll need configure this on a per-job basis, or in
`application_job.rb`, via [Active Job's `retry_on` API](https://edgeguides.rubyonrails.org/active_job_basics.html#retrying-or-discarding-failed-jobs).

## Wrapping Up
So far I'm feeling good about the simplification that this has afforded me/my apps, but I do find myself
missing the UI that Sidekiq provided, but [it does not seem that we'll have to wait too long for an
answer for that either](https://github.com/basecamp/solid_queue/issues/70).

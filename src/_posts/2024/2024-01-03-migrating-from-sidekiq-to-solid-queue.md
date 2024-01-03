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
I use HoneyBadger for error tracking, and I noticed that some jobs were not finishing, but I
was not notified of any errors. It turns out that this was due to [the default way that Solid Queue reports
errors](https://github.com/basecamp/solid_queue#other-configuration-settings), which is via the relatively
new [Rails Error Reporter](https://guides.rubyonrails.org/error_reporting.html).

It turns out that Honeybadger by default hooks into the error reporter, but for some reason it does
not report errors that report `handled: false`. I was only about to find this out after I did a bit of testing in the
Rails console:

```bash
irb(main):003> Rails.error.report(NotImplementedError.new("asdf"), source: "solid_queue", severity: :error, handled: false)
=> nil
irb(main):004> Rails.error.report(NotImplementedError.new("asdf"), source: "solid_queue", severity: :error, handled: true)
I, [2024-01-03T10:43:10.419883 #2]  INFO -- honeybadger: ** [Honeybadger] Reporting error id=090aa3b3-bd6e-482f-978a-3abacb5aa91e level=1 pid=2
irb(main):005> I, [2024-01-03T10:43:10.542319 #2]  INFO -- honeybadger: ** [Honeybadger] Success ⚡ https://app.honeybadger.io/notice/090aa3b3-bd6e-482f-978a-3abacb5aa91e id=090aa3b3-bd6e-482f-978a-3abacb5aa91e code=201 level=1 pid=2
=> nil
```

So armed with that information I explicity defined how errors should be reported in `production.rb`:

```ruby
 # Use a real queuing backend for Active Job (and separate queues per environment).
  config.active_job.queue_adapter = :solid_queue
  config.solid_queue.silence_polling = true

  config.solid_queue.on_thread_error = ->(exception) {
    Rails.error.report(exception, source: "solid_queue", severity: :error) # by default handled is set to `true`
  }
```

### Failures and Retries
By default, [Sidekiq has a built-in retry mechanism](https://github.com/sidekiq/sidekiq/wiki/Error-Handling),
but [this is not the case for Solid Queue](https://github.com/basecamp/solid_queue#failed-jobs-and-retries).
If automatic retrying is important for you you'll need configure this on a per-job basis, or in
`application_job.rb`, via [Active Job's `retry_on` API](https://edgeguides.rubyonrails.org/active_job_basics.html#retrying-or-discarding-failed-jobs).

## Wrapping Up
So far I'm feeling good about the simplification that this has afforded me/my apps, but I do find myself
missing the UI that Sidekiq provided, but [it does not seem that we'll have to wait too long for an
answer for that either](https://github.com/basecamp/solid_queue/issues/70).

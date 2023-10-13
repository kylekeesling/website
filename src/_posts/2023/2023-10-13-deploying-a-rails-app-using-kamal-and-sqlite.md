---
layout: post
title:  Deploying a Rails App Using Kamal and SQLite
tags: rails, ruby, stripe
date:   2023-10-13 00:00:00 -0400
categories:
  - rails
  - sqlite
  - litestack
  - kamal
blurb: |
  A couple of tips to greatly reduce the friction in getting a Rails app deployed.
---

There's been a lot of excitement lately in the Rails community. Rails World just wrapped up, which
brought with it some really great annoucements for the future of the platform, as well as the release
of [Rails 7.1](https://rubyonrails.org/2023/10/5/Rails-7-1-0-has-been-released).

This most recent release had a few particular additions/improvements that really piqued my interest
in regards to how I approach deployment of new Rails apps, specifically around the inclusion of a
[standard Dockerfile](https://github.com/rails/rails/blob/main/railties/lib/rails/generators/rails/app/templates/Dockerfile.tt) as well as a focus on making [SQLite](https://www.sqlite.org/index.html) a
first-class citizen/consideration for production apps.

When you combine that with my recent discovery of the [litestack gem](https://github.com/oldmoe/litestack),
the recent release of [Kamal](https://kamal-deploy.org/),
and the [recent GoRails series](https://gorails.com/series/build-a-url-shortener-with-rails-7)
on building your own URL shortner, and the stars had aligned.

## Prerequisites
Before we continue, here are a few things you'll want to make sure you've got setup/configured properly:

- Your app is using Rails 7.1 and has the default Dockerfile available to it
- [Docker](https://www.docker.com/get-started/) is installed on your local computer
- You have an account with a hosting provider like [DigitalOcean](https://www.digitalocean.com/) or
[Hetzner](https://www.hetzner.com/)

## SQLite + Litestack

For this project I wanted to host my project on the cheapest DigitalOcean droplet possible,
but it included some advanced requirements/features, including ActiveJob and ActionCable for background tasks and Turbo Broadcasts. While this certainly could have been done in the more traditional way of installing a
traditional database like PostgreSQL or MySQL and redis on the server, my goal was simplicity.

In contrast, leveraging a single technology like SQLite for all of these core aspects of the app would
allow me to greatly simplify the hosting requirements, which is where Litestack comes in.

If you aren't familiar, [Litestack](https://github.com/oldmoe/litestack) is a wonderful project with the following objective:

> Litestack is a Ruby gem that provides both Ruby and Ruby on Rails applications an all-in-one solution for web application data infrastructure. It exploits the power and embeddedness of SQLite to deliver a full-fledged SQL database, a fast cache , a robust job queue, a reliable message broker, a full text search engine and a metrics platform all in a single package.

Installing the gem is very straightforward, drop it in your `Gemfile` and run the installation command
`rails generate litestack:install`.

## Kamal

I'll admit it, I'm late to the Docker party, so there were many of the concepts I was (and am still)
unfamiliar with, but the abstraction Kamal provides and the default Rails `Dockerfile` are _just_
enough to allow me to get the job done.

That being said I do think there's value in more broadly understanding Docker as well as things
like Traefik, which is something that's mentioned many times in Kamal's docs, but is never really
explained. I can piece together that it's an orchestration and monitoring layer for your Docker
containers, but I've found it difficult to learn much more about it and how to use it, so if you
know of any good resources on better understanding it, please share.

The main gotcha I ran into was around how to store the SQLite databases in persistant storage.
I solved this by configuring a [Kamal volume](https://kamal-deploy.org/docs/configuration#using-volumes)
and adding the following environmental variable to my `.env` file: `LITESTACK_DATA_PATH=./storage`.

After a decent amount of trial and error, and losing my database between deploys more than once 😆,
the following deployment config is what ended up getting the job done.

```yml
# deploy.yml
service: url-shortner

image: kylekeesling/url-shortner

volumes:
  - "storage:/rails/storage"

servers:
  web:
    hosts:
      - 192.241.138.163
    options:
      "add-host": host.docker.internal:host-gateway
    labels:
      traefik.http.routers.rails_recipes.entrypoints: websecure
      traefik.http.routers.rails_recipes.rule: Host(`links.passtesting.com`)
      traefik.http.routers.rails_recipes.tls.certresolver: letsencrypt

registry:
  username: kylekeesling
  password:
    - KAMAL_REGISTRY_PASSWORD

env:
  clear:
  secret:
    - RAILS_MASTER_KEY
    - LITESTACK_DATA_PATH

traefik:
  options:
    publish:
      - "443:443"
    volume:
      - "/letsencrypt/acme.json:/letsencrypt/acme.json"
  args:
    entryPoints.web.address: ":80"
    entryPoints.websecure.address: ":443"
    entryPoints.web.http.redirections.entryPoint.to: websecure
    entryPoints.web.http.redirections.entryPoint.scheme: https
    entryPoints.web.http.redirections.entrypoint.permanent: true
    entrypoints.websecure.http.tls: true
    entrypoints.websecure.http.tls.domains[0].main: "links.passtesting.com"
    certificatesResolvers.letsencrypt.acme.email: "kyle.keesling@passtesting.com"
    certificatesResolvers.letsencrypt.acme.storage: "/letsencrypt/acme.json"
    certificatesResolvers.letsencrypt.acme.httpchallenge: true
    certificatesResolvers.letsencrypt.acme.httpchallenge.entrypoint: web
```

## Credit and Thanks

I'd like to send out a huge thanks to [Greg Molnar](https://greg.molnar.io/blog/deploying-a-rails-app-with-kamal/) and [Erik Minkel](https://www.erikminkel.com/2023/09/29/using-kamal-to-host-multiple-apps-on-a-single-server/),
who both provided some great content to help me along the way.

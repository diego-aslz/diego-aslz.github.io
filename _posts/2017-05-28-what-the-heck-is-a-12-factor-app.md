---
layout: post
title: "What the heck is The 12-Factor App?"
category: posts
comments: true
---

If you have worked with Heroku, you probably have heard of 12-factor. You
probably have added the [rails_12factor](https://github.com/heroku/rails_12factor)
gem to your projects. But are you aware of what the heck is that?

I heard about 12-factor apps recently and I wasn't sure what it was all about.
I decided to investigate about it and, thanks to the
[official website](https://12factor.net)
and [this blog post](http://www.clearlytech.com/2014/01/04/12-factor-apps-plain-english/)
I now know a bit more about it and I'd like to share it.

In summary, the 12-factor methodology consists in 12 rules that improve an
app's quality and maintainability. According to their website, _any developer
building applications which run as a service_ should read about it.

Here I'll briefly describe each factor and how it is applied to a Rails app.

## #1 - Codebase
> One codebase tracked in revision control, many deploys.

I think this one is pretty self-explanatory and easy to follow. You should
have one codebase. It should be versioned.

## #2 - Dependencies
> Explicitly declare and isolate dependencies.

You should have a way to explicitly declare which libraries you depend on and
which versions should be used. If you use Rails, Bundler + Gemfile are there
for you!

## #3 - Config
> Store config in the environment.

Long story short, **don't** store passwords or URLs (database, Redis,
 Elasticsearch) in your code! **Don't** even put them in your version
control! It's a security flaw!

Use gems like [dotenv-rails](https://github.com/bkeepers/dotenv) to
manage configuration in your dev machine (or even in production,
if there's no other way) and, if you use Heroku, it
provides you a way to set environment variables through `heroku config`.

## #4 - Backing services
> Treat backing services as attached resources.

Your code should not depend on specific environment configurations to run.
If you switched from a local Redis server to an Amazon's Elasticache service,
your code should **not** have to change.

The approaches recommended in the previous item can be used here too.

## #5 - Build, release, run
> Strictly separate build and run stages.

Deploying new code to production should be easy and straightforward. A single
command in the terminal would be ideal. It should build, release and ensure
the new version is running.

In the Rails world, the tools used here are usually
[Capistrano](https://github.com/capistrano/rails) or
`git push heroku master` if you deploy to Heroku.

## #6 - Processes
> Execute the app as one or more stateless processes.

Your app is probably run by several servers. All processes and requests should
be stateless. In case one server fails or goes down, the other should be able
to handle a subsequent process or request without a problem.

**Twelve-factor processes are stateless and share-nothing.**

## #7 - Port binding
> Export services via port binding.

The services your app provides should be bound to a port, like 80 for HTTP
services. The case here is that in development you'll access your services,
for instance, via port 3000 and in production via port 80, but you **will**
use a port. By explicitly declaring a dependency like Puma (see #2), you'll
have that webserver library binding your service to a port. One app can have
multiple services bound to different ports. It can also be the backing service
to another app (see #4).

## #8 - Concurrency
> Scale out via the process model.

Your app should have small and well-defined processes to handle requests. You
can have background jobs here to do the heavy work, like sending out emails, so
you can return a response to the user as quickly as possible. This improves
scalability and allow you to scale your app by just adding more CPU or Memory
and have more workers running.

## #9 - Disposability
> Maximize robustness with fast startup and graceful shutdown.

The processes that run your app should be disposable, i. e. shutdown and start back up gracefully and quickly so you can deploy several times a day.
Heroku already gives you 0 downtime deploy, but you should make your app
load fast so you won't have requests waiting 20 seconds for your app to load.
If you use a VPS, solutions like Puma and Unicorn support 0 downtime deploy,
you should definitely look into it if you're not familiar.

## #10 - Dev/prod parity
> Keep development, staging, and production as similar as possible.

You should minimize the gap between your environments. Use in development the
same database you use in production, the same webserver library, etc. New code
should be deployed soon (a few hours tops). Developers should be able to deploy
without depending on ops engineers.

## #11 - Logs
> Treat logs as event streams.

_A twelve-factor app never concerns itself with routing or storage of its
output stream_. A 12-factor app should have logs streamed to the standard
output and they you should be available even in production environment.

Services like [Papertrail](https://papertrailapp.com/) can help you here.

## #12 - Admin processes
> Run admin/management tasks as one-off processes.

It's common to need to run one-off administrative or maintenance tasks, like
cleaning up data or extracting information for a report you are working on.
These processes should be run in an identical environment as the production
processes.

So, you shouldn't connect your local machine directly to the production
database and run tasks locally. If you use Heroku, `heroku run console`
will start up a one-off process that will have the same configuration your
production dynos have.

## Wrapping it up

In general, I find these rules easy to follow. You usually get most of them
for free. I bet most developers/engineers already follow them without even
knowing they are in a list called __The 12-Factor__ (my case).

So here you have it, hope you find this useful!

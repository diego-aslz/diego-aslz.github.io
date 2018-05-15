---
layout: post
title: "Recursive Queue"
category: posts
comments: true
---

In architectures based on microservices, message queues like RabbitMQ are
a common thing for communication. Even though exchanging messages is an elegant
solution, you're not free from eventually running into obstacles with not
so obvious solutions.

In this post, I want to talk about a solution I came up with and I found very
powerful and elegant. I'm sure other people have already thought of it and you
may have done it yourself, but I'll write about it anyway because it was
a bit mind-blowing for me. Hope it is for you too!

## The Problem

The obstacle we ran into was: long running message. We had this one kind of
message that was about scraping Products for a Category from a website.
It went all the way through hundreds of pages for the given Category
getting products out of them. This also means we depended on Internet
connection for the message to work. Another important detail was
that this message needed to know exactly when it finished to send yet
another message and alert everyone "Hey, I finished Category X!".
It took hours (sometimes days) to finish processing one Category.

What if we deployed while it was running? What if we got a connection
hiccup while scraping? Well, an exception would be raised, the consumer
would respond with a `nack` and the message would go back into the queue
to be picked up by the next consumer, which in turn had to start
**everything** all over.

This was bad for us. Hours of processing just wasted. The third-party
scraping solution also has quota limits we must respect and we couldn't
afford to just throw away thousands of requests each time someone deployed.

## The Solution

One idea we came about was to use a Redis server to save state, which
would be the number of the page being scraped. So after a failure,
we would read the page from Redis and start from there on.
It would always check Redis to see if a page was already being scraped
and continue from there if that was the case. After finishing,
the consumer would just remove any state from Redis so the next time
this Category was scraped we'd start from page 1.

This solution worked, but it didn't feel quite right. We would be communicating
by sharing state, not sharing state by communicating. It would also add
another reliability: Redis. Also, the implementation got a lot more complex.
It wasn't very cool.

Then an idea hit me, inspired on Elixir's "head & tail" approach. In case
you don't know the "head & tail" part (several languages support it), it is
about processing lists. Instead of looping through the items in the list, you
make the function/method recursive: it takes the head (first item) out of
the list, processes it and calls itself recursively with the tail (remaining
items). Once the method is called with an empty list, it means we got
to the end of the list.

So we did this: we changed the message to receive an extra argument:
current page number. If it's missing, we use 1. The consumer would then go
ahead, process **only** that page (the head) and before sending an `ack`,
schedule the same message to be processed again (recursively), but incrementing
the page number.

Messages would go like this:

* Message received: Category X, no page given
  * Process page 1 for Category X
  * Publish a new message: Category X, page 2
* Message received: Category X, page 2
  * Process page 2 for Category X
  * Publish a new message: Category X, page 3
* And so on and on...

When we got to the page after the last, trying to scrape it would give us a 404
response. That meant we finished processing Category X and we can safely
send the message "Hey, I finished Category X!"

If we happened to deploy while one of these message were running, we'd only
lose 1 page: the one being scraped at that time. An error would be raised
just the same, the message would go back into the queue, but since the
current page is in the payload, when it started again it would continue
from that page onwards.

There: no Redis, no shared state, elegant and powerful. We also got to delete
some loops and the implementation got a lot simpler!

Thanks Elixir! (And all the languages that inspired it)

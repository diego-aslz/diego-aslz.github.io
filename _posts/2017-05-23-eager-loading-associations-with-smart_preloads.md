---
layout: post
title: "Eager Loading Associations with smart_preloads"
category: posts
---

A few weeks ago I came across an issue while dealing with associations I had
to eager load.

The product I'm working on supports custom templates using 
[liquid](https://github.com/Shopify/liquid). That means our
customers can customize how their content is shown.

It's all nice and fun until you get lots of customers and tons of data.
To avoid N + 1 queries, our approach was to eager loaded some important
associations. It makes pages that use them a bit faster, but pages that
didn't use those associations were having an unnecessary overhead. Not
to mention the memory usage.

So, the issue was: some templates need certain associations to be loaded,
some don't need any association to be loaded at all.

## How can I make this dynamic?

It crossed our minds statically analyzing templates before rendering them
to find which associations were being used. But that wasn't for sure an
easy-to-implement approach and it'd be prone to flaw, so we discarded it.

Then I had an idea and, since I did not find any other gem solving this issue
the way we needed it to be solved, I decided to implement it myself.

I created my own list class, an Array decorator, to intercept calls to the
raw records collection. I also created an Item class to intercept calls to
the record itself.

This way, I could rewrite the methods that were accessing associations and
ask the custom list to preload that association for all items in the array
before following the call to the original method.

The result is that, now, associations are only loaded **if** they are used
in code! The memory usage and page load were both improved!

In the images below, you can see our Response Time, Memory Usage and Dyno
Load (we use Heroku) respectively before and after the deploy (`v892` 
marks the deploy).

![Response Time](/images/smart_preloads/response_time.png "Response Time")
![Memory Usage](/images/smart_preloads/memory.png "Memory Usage")
![Dyno Load](/images/smart_preloads/dyno_load.png "Dyno Load")

## The Gem

The next thought was: for sure more people can benefit from this.

Then I went ahead and created a gem, extracted my code and released it. The 
gem is [smart_preloads](https://github.com/nerde/smart_preloads).

Read the docs if you'd like to use it, here is an example of how associations
are loaded as they are used:

{% highlight ruby %}
@authors = Author.all.smart_preloads

@authors.each do |author|
  puts author.name
end
#=> SELECT "authors".* FROM "authors"

@authors.each do |author|
  author.books.each do |book|
    puts "#{author.name} authored #{book.name}"
  end
end
#=> SELECT "books".* FROM "books" WHERE "books"."author_id" IN (1, 2)
{% endhighlight %}

Notice that all inner associations are also wrapped by the custom list class, so nested associations are loaded all at once too!

Thanks for reading. Hope you find this useful!

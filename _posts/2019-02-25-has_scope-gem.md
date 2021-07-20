---
layout: post
title: "The has_scope Gem"
category: posts
comments: true
---

<iframe width="560" height="315" src="https://www.youtube.com/embed/ovItZv_1doI" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

In this episode I’m gonna show you a gem that I find quite useful to clean up
controllers and not many developers know about. It’s the `has_scope` gem.

To illustrate its purpose, I have here a simple Rails app with a products page.
In this page the user can search products by name, price and also navigate through
pages using pagination.

Now I’m gonna show you the code in the controller.

Here you can see we check for each different filter that may be applied and also
apply pagination after that. This is not ideal, controllers should be thin because
their code is not reusable and fat controllers are hard to maintain.

Let’s remove all the code responsible for building conditions from the controller
and move it to the Product model, where it should be.

Notice that for the price filter I’m creating a single scope. This is to
illustrate a different  scenario later on. Keep in mind that this is a
contrived example and I would keep these filters in separate scopes in a real application.

Cool, now our controller is a bit cleaner and we can start using the gem. First,
let’s add it to our Gemfile and run `bundle` to install it. Remember to
restart your Rails server after this.

With the gem installed, we can reduce the code in this controller even further.

The gem’s purpose is to apply this kind scopes for us. To do that, we simply need to
call `has_scope` and give it some configuration options. The first scope is the
simpler one because we want the `search` parameter to be mapped to the `search`
scope. Therefore, all we need to do is to add `has_scope :search` , call
`apply_scopes` in the action and get rid of the manual `@products.search` call.

Keep in mind that the gem will only apply the filter if the parameter is present,
which is what we want. If the parameter is missing, the scope will not be called.

Now we have the price filter, which is a bit more complicated.
The `price` parameter is a Hash, but the scope takes two arguments instead.
To handle this, we need to use `type: :hash, using: [:min, :max]`. The gem will
use the keys in the array to get those values out of the `price` parameter and
pass them down to the scope in that order.

Also, the parameter we’re getting is called `price`, but the scope is called `by_price`.
To tackle that, `has_scope` allows us to customize the parameter name with the
`as` key.

Now we can just drop the price filter calls from the action.

We have one more scope to get rid of. This is an easy one. The only difference
here is that we want the `page` scope applied all the time, regardless of
the parameter being there or not.

To achieve that, we just add `has_scope :page, default: 1`. When you give
`has_scope` a default value, it will always applied.

Then we can remove the `page` call and it's done.

So here you have it: a cleaner controller and reusable scopes. The gem
provides a few more features I haven't covered in this episode. Make sure
you check out their docs if you're interested.

Thank you for watching and I hope you enjoyed this snack!

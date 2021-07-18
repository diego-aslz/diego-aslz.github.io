---
layout: post
title: "My Thoughts on Error Handling"
category: posts
comments: true
---

<iframe width="560" height="315" src="https://www.youtube.com/embed/QQNAsANS-ZA" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

Exceptions are tricky. Most people don't give them the attention
they deserve and I think it's because most of the time they don't really know what to do
about them.

In this episode I want to talk about some good practices that I've learned about handling
errors and things you should avoid to save yourself from some headaches.

Let's start with simple things that you should not do. Here we have a variable that might or might not be `nil`.

```ruby
product = nil

product.name rescue nil
product.name rescue ''
```

I usually see people writing this kind of code when they want to avoid getting a `NoMethodError` when
`product` is nil. Please don't do this. The reason this is bad is because that rescue will swallow any
exception you throw at it. If you have a typo in there, for example, rescue will catch it and it'll easily
go unnoticed.

```ruby
produt.name rescue nil
```

The right way to handle this scenario would be to `try!` and, in the second example, calling `to_s`
if you really want an empty String when either `product` or `name` is `nil`. Remember you need ActiveSupport
for `try` to be available.

```ruby
require 'active_support/all'

product.try!(:name)
product.try!(:name).to_s
```

`try!` will just return `nil` if `product` is `nil`. Otherwise, it will call the method. Be aware that the bang
version of `try` will raise an error if `name` is not a valid message for `product`. If you don't want that, you have
to go with simple `try`, no bang.

```ruby
product.try(:name)
product.try(:name).to_s
```

Since Ruby 2.3.0 you can also use safe navigation, which makes this code even simpler

```ruby
product&.name
product&.name.to_s
```

Safe navigation has the same effect as the `try!` here.

Now take a look at this simple controller action.

```ruby
def create
  Product.create!(product_attributes)
rescue
  flash[:error] = 'Failed to create Product'
  redirect_to action: :index
end
```

This is trying to create a product with the bang version of `create`, which fails if any validation fails.
Then we rescue from anything, add a generic flash message and redirect the user to another page.

There are several things wrong with this code. First, we should not rescue from anything. Nor should we rescue from
`Exception` like this, which all exceptions inherit from

```ruby
def create
  Product.create!(product_attributes)
rescue Exception => _
  flash[:error] = 'Failed to create Product'
  redirect_to action: :index
end
```

In Ruby all errors you can get inherit from `Exception`, and that means rescuing from
`Exception` will rescue from everything, even the errors you don't need or don't really want to rescue from,
like `SyntaxError` and `Interrupt`. By rescuing `Interrupt`, for example, you're
basically preventing the user from hitting `Ctrl + C` while inside this block, which is very bad.

The right approach here is to be specific. What is it that you're rescuing from? Can you know exactly which
exceptions you're looking for? Maybe `ActiveRecord::RecordInvalid`?

```ruby
def create
  Product.create!(product_attributes)
rescue ActiveRecord::RecordInvalid => _
  flash[:error] = 'Failed to create Product'
  redirect_to action: :index
end
```

If you don't know which errors to expect and want to really rescue from anything bad that can happen to your
code, use `StandardError`. `StandardError` is the base class for common runtime errors.

```ruby
def create
  Product.create!(product_attributes)
rescue StandardError => _
  flash[:error] = 'Failed to create Product'
  redirect_to action: :index
end
```

By the way, when creating your own error classes, inherit from `StandardError`, NOT from `Exception`.

```ruby
class MyCustomError < Exception # change to StandardError
end
```

Now to the next issue: we are hiding the problem. We are redirecting the user to a page with a generic error
message and completely ignoring the error that occurred. Good luck tracking down any bug when the user
comes back to you saying "Hey, I'm getting this Failed to create Product message" and you have nowhere to look
at to see the error. No logs, no bug tracking system.

You should not do this, don't hide errors. If the system fails, let it fail and show the user a 500 error
page and that's it. Let the error go to your bug tracking system or get dumped to your logs so you can look at it.

One could argue that instead we could add the error message to the flash message we're gonna show the user.

```ruby
def create
  Product.create!(product_attributes)
rescue StandardError => err
  flash[:error] = "Failed to create Product: #{err.message}"
  redirect_to action: :index
end
```

I disagree. You don't want your users to see things like `Unable to connect to database XYZ with user ABC` or
other technical details that might expose your system to attackers. Unless we're talking about errors we know are
safe, like validation errors, then it should not be exposed to end users. Expose it somewhere developers can see
it, not users.

Another kind of scenario that I see often is this. We explicitly raise an error if validation fails here.

```ruby
def create
  if Product.create(product_attributes)
    redirect_to action: :index
  else
    raise 'Failed to create Product'
  end
end
```

This is bad because validations are expected to fail if users provide invalid data. Unless you're rescuing
this somewhere, this error will get out in the logs or into your bug tracking system, and there's no point in doing that.
There's nothing you can do about it here, the user did something wrong and you should show him the problem
and consider the request fulfilled, no errors necessary. I've seen bug tracking systems get filled in with
errors that could be dismissed like this one and it ends up hiding real problems.

This brings me to a few rules that I always apply when dealing with errors in backend systems:

1. If it's a technical error, the client should get a generic 'We screwed up' message and the error goes to
  bug tracking system
2. If it's a client error like invalid user provided data, then the client gets a very specific error message
  and the bug tracking system gets nothing

And the last thing about errors is: where should you be handling them?

The answer is: highest level possible. For example: in the controller, rake task or job that you're executing.
Why? Because that makes your low level code reusable. Very often error handling means notifying bug tracking
systems, or sending out an email or printing out to the logs. You shouldn't do that in lower levels of your code
because you might want the same functionality to be executed from two different places and error handling to be
different for each of them.

And that's it for this episode. Thank you for watching and I hope you enjoyed this snack!

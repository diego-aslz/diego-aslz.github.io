---
layout: post
title: "Pessimistic Locking with ActiveRecord"
category: posts
comments: true
---

<iframe width="560" height="315" src="https://www.youtube.com/embed/MOcdL3F6NyM" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

There are situations where you need to make sure concurrent processes that could potentially be changing the
same record don't cause data inconsistencies.

Let's take a look at one example. In this system I have a naive and contrived implementation of a banking system.
We have accounts with balance:<- HERE

```ruby
account = Account.first
# => #<Account:0x00007f789427af88 id: 1, owner: "Diego", balance: 50>
```

Let's say we have two processes changing this account's balance. The first one wants to add $25 and the second one
wants to withdraw $10. Given that account has $50 in balance, the end result should be $65.

We're going to simulate this with two IRB sessions connecting to the same database

The first process loads the account:

```ruby
account = Account.find(1)
```

Then the second process loads it as well:

```ruby
account = Account.find(1)
```

Now the first one changes the balance:

```ruby
account.balance += 25
account.save
```

At this point, the second process has already loaded the account in memory. As you can see, it still has $50
in balance:

```ruby
account
# => #<Account:0x00007f789427af88 id: 1, owner: "Diego", balance: 50>
```

If it proceeds and withdraws $10 from the account, here's what happens:

```ruby
account.balance -= 10
account.save

account.reload
# => #<Account:0x00007fea155337a0 id: 1, owner: "Diego", balance: 40>
```

As you can see, the balance is now $40 instead of the $65 we expected because the second process saved for last.
The moment the first process saved the record, the balance in the database was $75, but the second process had already
loaded the account with $50 in balance. Its in memory account became stale, which means it didn't have the most
up to date information anymore. Then it changed the balance to $40 and saved that, ignoring the change
from the first process.

To solve this, we need to prevent processes from working with stale data. Some sort of locking is
usually the solution for this. We can lock the record to prevent concurrent updates.

ActiveRecord provides two kinds of locking: pessimistic and optimistic. Pessimistic locking uses database locks
to prevent simultaneous access to a specific table row. Optimistic locking uses a database field to control
the record version, preventing a process from updating a stale object. In this episode I'll explain how
pessimistic locking works and I will cover optimistic locking in the next one.

Let's change our code to grab a database lock on the record we're trying to update.

First, we need to restart our exercise with a 50-dollar account. Let's load it in the first process.

```ruby
account = Account.find(1)
```

And in the second:

```ruby
account = Account.find(1)
```

Now, when making changes, we'll call `with_lock`, like this:

```ruby
account.with_lock do
  account.balance += 25
  account.save
end
# TRANSACTION (0.2ms)  BEGIN
# Account Load (0.3ms)  SELECT "accounts".* FROM "accounts" WHERE "accounts"."id" = $1 LIMIT $2 FOR UPDATE  [["id", 1], ["LIMIT", 1]]
# Account Update (0.2ms)  UPDATE "accounts" SET "balance" = $1 WHERE "accounts"."id" = $2  [["balance", 75], ["id", 1]]
# TRANSACTION (6.0ms)  COMMIT
```

As you can see, the `with_lock` call created a transaction and reloaded the account. You can see there is a `SELECT`
statement in there pulling the account from the database again, even though it was already in memory. You can also
notice there's a `FOR UPDATE` modifier at the end of the `SELECT` statement. That's how you lock a record in
Postgres, which is the database I'm using, and ActiveRecord abstracts this for us. Until this transaction
is committed or rolled back, that particular account cannot be selected for update by another transaction.
This prevents concurrent access to the account.

Then the rest of the statements are straightforward. We update the account and commit the transaction.

At this point, we know the second process has stale data in memory:

```ruby
account
# => #<Account:0x00007f789427af88 id: 1, owner: "Diego", balance: 50>
```

As you can see, its balance is still $50. So let's run its update again, but using `with_lock`:

```ruby
account.with_lock do
  account.balance -= 10
  account.save
end
# TRANSACTION (0.2ms)  BEGIN
# Account Load (0.4ms)  SELECT "accounts".* FROM "accounts" WHERE "accounts"."id" = $1 LIMIT $2 FOR UPDATE  [["id", 1], ["LIMIT", 1]]
# Account Update (0.3ms)  UPDATE "accounts" SET "balance" = $1 WHERE "accounts"."id" = $2  [["balance", 65], ["id", 1]]
# TRANSACTION (6.1ms)  COMMIT
```

As you can see, the account was again reloaded from the database right after starting the transaction.
The `FOR UPDATE` modifier is here again to tell the database to lock this record so no other transaction
can lock it. This guarantees both processes are working with an up to date object. We can confirm the
balance is correct now by reloading the account:

```ruby
account.reload
# => #<Account:0x00007fc5e1cb8b40 id: 1, owner: "Diego", balance: 65>
```

If both processes run at the same time, the database will only allow one of them to access the record at a time.
To see that, let's restart the exercise and this time make the first process sleep for a while before returning,
so we have time to kick off the second one.

```ruby
account.with_lock do
  10.times do
    puts 'Doing some heavy computation'
    sleep 1
  end
  account.balance += 25
  account.save
end
```

And in the second process, let's run the update without sleeping.

```ruby
account.with_lock do
  account.balance -= 10
  account.save
end
```

The second process is frozen until the first one commits its transaction, releasing the lock.

This may be a good enough solution for some scenarios, but it has some down sides. First, you _do_
have to remember to call `with_lock` everywhere you intend to update the account's balance.

This account update is straightforward, but you might have scenarios in which you would, for example, only
conditionally update the record and, in that case, you might end up unnecessarily locking the record.

This solution also has scalability issues:

* You might hold a record locked for longer than you need if you do more work within the transaction
* If you have high traffic, you'll get database lock contention if many requests try to grab a lock to update
  the same record. One process will execute at a time, the others will be kept waiting
* You can easily get deadlocks if you have transactions competing for the same resources

In the next episode I'll show you a better locking solution that's also built into ActiveRecord and
scales a lot better.

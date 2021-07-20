---
layout: post
title: "Better Queries with Arel"
category: posts
comments: true
---

<iframe width="560" height="315" src="https://www.youtube.com/embed/Nvt0FnIaMBc" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

ActiveRecord is a very powerful tool. It goes a long way allowing you to organize your database
access and build queries in the Ruby way. However, in some scenarios we end up having to write
SQL statements in Strings because ActiveRecord just won't generate the SQL we need for us.

Under the hood, ActiveRecord uses a tool called Arel to build SQL statements. We can leverage
that to build our own advanced queries.

Before we dive into coding mode, be aware that there are gonna be dozens of code examples in this episode.
Feel free to pause the video to read and understand them thoroughly. You can also see those examples in
the repository I'll link in the description.

Let's get to it. We have here two simple models with a simple relationship: Category and Product.
Product belongs to Category. Now let's jump to the Rails console where I can show you some queries we
can build with Arel using these two models.

Every ActiveRecord model class has a method called `arel_table`.

```ruby
Product.arel_table
# => #<Arel::Table:0x00007fd18cbb1ce0
#  @name="products",
#  @table_alias=nil,
#  @type_caster=#<ActiveRecord::TypeCaster::Map:0x00007fd18cbb1d30 @types=Product(id: integer, name: string, price: float, active: boolean, created_at: datetime, updated_at: datetime)>>
```

This method returns an object representing that model's table in the database. With
that object we can access columns like this:

```ruby
Product.arel_table[:id]
# => #<struct Arel::Attributes::Attribute
#  relation=
#   #<Arel::Table:0x00007fd18cbb1ce0
#    @name="products",
#    @table_alias=nil,
#    @type_caster=#<ActiveRecord::TypeCaster::Map:0x00007fd18cbb1d30 @types=Product(id: integer, name: string, price: float, active: boolean, created_at: datetime, updated_at: datetime)>>,
#  name=:id>

Product.arel_table[:name]
# => #<struct Arel::Attributes::Attribute
#  relation=
#   #<Arel::Table:0x00007fd18cbb1ce0
#    @name="products",
#    @table_alias=nil,
#    @type_caster=#<ActiveRecord::TypeCaster::Map:0x00007fd18cbb1d30 @types=Product(id: integer, name: string, price: float, active: boolean, created_at: datetime, updated_at: datetime)>>,
#  name=:name>
```

With those columns we can build conditions. Let's see an example.

Look at this query. We are using a raw string here because ActiveRecord doesn't
provide us a way to do this in the object oriented manner. But Arel does!

```ruby
Product.where('name ILIKE ?', '%shoe%')
# SELECT "products".* FROM "products" WHERE (name ILIKE '%shoe%')
```

We can use `matches`

```ruby
Product.where(Product.arel_table[:name].matches('%shoe%'))
# SELECT "products".* FROM "products" WHERE "products"."name" ILIKE '%shoe%'
```

What's the benefit? Well, for starters, Arel took care of quoting the column name for us and also prefixed it
with the table name to avoid ambiguous matches. Also, now we're not writing SQL inside a ruby file anymore and if
we switch database providers, the new adapter will take care of generating the right SQL for us. Keep in mind
that `ILIKE` is not the right keyword for every database.

Moving on. You can build several different conditions with Arel. We have methods for =, IN, >, >=, <, <=.

```ruby
Product.where(Product.arel_table[:name].eq('shoe'))
# SELECT "products".* FROM "products" WHERE "products"."name" = 'shoe'

Product.where(Product.arel_table[:name].in(['shoe', 'sneakers']))
# SELECT "products".* FROM "products" WHERE "products"."name" IN ('shoe', 'sneakers')

Product.where(Product.arel_table[:price].gt(100))
# SELECT "products".* FROM "products" WHERE "products"."price" > 100.0

Product.where(Product.arel_table[:price].gteq(100))
# SELECT "products".* FROM "products" WHERE "products"."price" >= 100.0

Product.where(Product.arel_table[:price].lt(100))
# SELECT "products".* FROM "products" WHERE "products"."price" < 100.0

Product.where(Product.arel_table[:price].lteq(100))
# SELECT "products".* FROM "products" WHERE "products"."price" <= 100.0
```

Some of them have their negative forms like != and NOT IN.

```ruby
Product.where(Product.arel_table[:name].not_eq('shoe'))
# SELECT "products".* FROM "products" WHERE "products"."name" != 'shoe'

Product.where(Product.arel_table[:name].not_in(['shoe', 'sneakers']))
# SELECT "products".* FROM "products" WHERE "products"."name" NOT IN ('shoe', 'sneakers')
```

There are many predications you can use with arel columns. To get a list of them all, run
`Arel::Predications.instance_methods`. This list varies depending on the Rails version you're using.

We can also combine conditions together. If you use `where` clauses from ActiveRecord you know they are gonna
generate you `AND` conditions.

```ruby
Product.where(Product.arel_table[:name].eq('shoe')).where(Product.arel_table[:id].eq(1))
# SELECT "products".* FROM "products" WHERE "products"."name" = 'shoe' AND "products"."id" = 1
```

If you need an `OR` condition, you can do that with Arel like this.

```ruby
Product.where(Product.arel_table[:name].eq('shoe').or(Product.arel_table[:id].eq(1)))
# SELECT "products".* FROM "products" WHERE ("products"."name" = 'shoe' OR "products"."id" = 1)
```

You can even combine ANDs and ORs.

```ruby
name = Product.arel_table[:name]
Product.where(name.eq('shoe').or(Product.arel_table[:id].eq(1).and(name.matches('%sneakers%'))))
# SELECT "products".* FROM "products" WHERE ("products"."name" = 'shoe' OR "products"."id" = 1 AND "products"."name" ILIKE '%sneakers%')
```

Now let's get fancy with functions. Have you ever had to call a database function from your Rails code? Arel can help you there!

```ruby
Product.where(Arel::Nodes::NamedFunction.new('LENGTH', [Product.arel_table[:name]]).gteq(50))
# SELECT "products".* FROM "products" WHERE LENGTH("products"."name") >= 50
```

The objects we are passing to the `where` calls in all these examples are `Arel::Nodes`.

```ruby
Arel::Nodes::NamedFunction.new('LENGTH', [Product.arel_table[:name]]).gteq(50).class
# => Arel::Nodes::GreaterThanOrEqual

Product.arel_table[:name].eq('shoe').class
# => Arel::Nodes::Equality
```

Nodes can be used in `select` and `order` calls too!

```ruby
Product.select(Arel::Nodes::NamedFunction.new('LENGTH', [Product.arel_table[:name]]))
# SELECT LENGTH("products"."name") FROM "products"

Product.order(Arel::Nodes::NamedFunction.new('LENGTH', [Product.arel_table[:name]]))
# SELECT "products".* FROM "products" ORDER BY LENGTH("products"."name")
```

In case of ordering, you can also call `desc` on a node to reverse it:

```ruby
Product.order(Arel::Nodes::NamedFunction.new('LENGTH', [Product.arel_table[:name]]).desc)
# SELECT "products".* FROM "products" ORDER BY LENGTH("products"."name") DESC
```

The next feature we're gonna see allows us to create better joins. Have you ever had to do something like
this to get a `LEFT OUTER JOIN`?

```ruby
Category.joins('LEFT OUTER JOIN products ON products.category_id = categories.id')
# SELECT "categories".* FROM "categories" LEFT OUTER JOIN products ON products.category_id = categories.id
```

Arel can do this for us too!

```ruby
categories = Category.arel_table
products = Product.arel_table
Category.joins(
  categories.create_join(
    products,
    products.create_on(categories[:id].eq(products[:category_id])),
    Arel::Nodes::OuterJoin)
)
# SELECT "categories".* FROM "categories" LEFT OUTER JOIN "products" ON "categories"."id" = "products"."category_id"
```

You can also combine conditions here:

```ruby
Category.joins(
  categories.create_join(
    products,
    products.create_on(categories[:id].eq(products[:category_id]).and(products[:name].not_eq(nil))),
    Arel::Nodes::OuterJoin)
)
# SELECT "categories".* FROM "categories" LEFT OUTER JOIN "products" ON "categories"."id" = "products"."category_id" AND "products"."name" IS NOT NULL
```

Now that you've learned Arel, let's use what we saw here to refactor some scopes that I added to our models.

First let's improve these price filters to use Arel.

Now for the `search` scope we'll convert it into a class method because it's gonna get bigger. Remember: any
class level methods in an ActiveRecord model can be used as a scope.

Now it's time for the `active` scope.

In this case in particular, I'm going through another model's arel table. Usually when I see that happening,
either with `arel_table` or just `where` clauses I recommend creating a scope inside that model, in this
case: `Category`. As you can see here we already have an `active` scope inside category. How can we reuse
it in the `Product` model? Simple, we just use another ActiveRecord trick: the `merge` method.

This method allows us to combine together conditions from two different ActiveRecord relations.

And there you have it: no more raw SQL in our models.

I know this code is verbose and a lot longer than the previous version using a String, but it's a lot more
reusable and high level. If the verbosity bothers you, take a look at the gem called `arel_helpers`.
It provides several methods to use Arel with less code.

And that's it for this episode. Thank you for watching and I hope you enjoyed this snack!

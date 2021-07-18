---
layout: post
title: "Behavior Driven Development with Rails"
category: posts
comments: true
---

[![Behavior Driven Development with Rails on Youtube](https://img.youtube.com/vi/U73P7dbyqGg/0.jpg)](https://www.youtube.com/watch?v=U73P7dbyqGg)

In the previous episode, I explained how I like to test my Rails apps with an example. In this episode, I want to take that example one step further and rewrite it with Cucumber.

Cucumber is a tool that allows us to implement Behavior Driven Development. It allows developers to define high level scenarios that describe the behavior of a given feature in a way that a stakeholder or other non-tech people can understand more easily.

I'm not gonna show you how to install the Cucumber gem in your Rails app, but I'm linking it down in the notes, you can follow the steps there.

So without any further ado, let's get to it.

Here are the two test cases from the last episode:

```ruby
# spec/features/customer_spec.rb
describe 'Customer Management' do
  scenario 'Creating a new customer' do
    visit new_customer_path

    fill_in 'First Name', with: 'Diego'
    fill_in 'Last Name', with: 'Selzlein'
    click_on 'Save'

    expect(page).to have_content('Diego Selzlein')
    customer = Customer.last
    expect(customer.first_name).to eq('Diego')
    expect(customer.last_name).to eq('Selzlein')
  end

  scenario 'Trying to create an invalid customer' do
    visit new_customer_path

    click_on 'Save'

    expect(page).to have_content('First Name is required')
    expect(page).to have_content('Last Name is required')
  end
end
```

I've created an app with a basic scaffold for Customer management. We're going to test it with Cucumber.

While you can write Rspec tests in pure Ruby, Cucumber uses another language for defining them: Gherkin.

Gherkin is a high level and easy to understand language. It's focused on describing behavior. We'll rewrite these two tests to learn it.

In our project, let's create a feature file for these tests under `features`. Let's call it `customer_management.feature`.

The first line in a feature file gives a name to the feature being tested. It has to start with `Feature:`, the rest of the line is free text. We're going to call this feature `Customer Management`.

Now we'll define our first scenario: `Creating a new customer`. It has to start with `Scenario:`, like this. This is equivalent to an `it` call in Rspec. It groups a set of steps that compose a feature. We have to define now which steps compose the feature to create a customer.

Steps have to start with either `Given`, `And`, `But`, `When` or `Then`. Which one you choose makes no difference as far as Cucumber is concerned, these words are just for readability.

Now think about what needs to be done in the system. I can think of two steps:

1. Create a customer
2. Make sure it was persisted

Let's go with that. First, let's define a `When` step that creates a customer. `When` is usually used when defining the action being tested. The rest of the line is arbitrary.

This step has a table right below it. This is another Gherkin feature and Cucumber will parse that table for us. You'll see later how to use it.

Next, we define a `Then` step, which checks if that customer we just created has been added to the system. Again, besides the prefix, the rest of the line is arbitrary. This step also takes a table, which we'll use later to match against the database and make sure the customers in the Gherkin table match the ones we have stored.

When writing these steps, choose words that an end user would understand. Avoid technical terms, that would couple your feature definition with implementation details. Notice that we're not mentioning browser, page, URL, input, nothing technical. This means that we can completely change the interface from a browser page to, let's say, a command line that performs the same action and this feature description would not need to change.

```gherkin
Feature: Customer Management

  Scenario: Creating a new customer
    When I create the following customer:
      | First Name | Diego    |
      | Last Name  | Selzlein |
    Then I should have the following customers:
      | first_name | last_name |
      | Diego      | Selzlein  |
```

Now let's run this and see what happens. For that, we have to execute `bundle exec cucumber`.

As you can see in the output, Cucumber found our feature definition and tried to do something with it. In fact, it tried to execute it.

For each step in our feature specification, Cucumber will try to find a block of code to execute that performs that step. However, the 2 steps we have in this file are not defined yet, so Cucumber tells us they're pending and gives us a code snippet to define those steps.

Let's copy this code and paste it in a steps file, which will be in `features/step_definitions/customer_steps.rb`.

The first step definition will run when Cucumber executes the line `When I create the following customer:`. You can see it takes a table as an argument. This is the table we defined in the feature file. We get it parsed into an object and we can use it here to execute this step.

Since this one is about creating a customer, we want to visit the new customer page, fill it in with the details we got from that table object and submit it. Let's borrow the code from the Rspec definition and adjust it.

In Rspec, we were using a static value for each field. In here, we want to use whatever is in the table definition. For that, we'll call `table.rows_hash`, which transforms a 2-column table into a `Hash` object, mapping the first column with the second. In our table, it'll be a `Hash` like this.

Now, all we need to do is to use that in the step execution.

Let's run Cucumber once again and see how it looks like.

As you can see, the first step now is green, which indicates it executed perfectly. It found the form and the fields, filled them in and submitted. Now the second step is to verify we actually persisted that customer. Let's implement it.

If we look again at the step definition, we see there's a table parameter here as well. We need to match it against the database customers table and make sure it has the right data. There's another handy method that these table objects provide, it's the `diff!` method. It matches a table against an Array of Hashes and raises an error if they do not match.

So all we have to do here is to call `table.diff!` and then get an Array of Hashes with the customers we have in the database to run the diff against. We can get that with `Customer.all.map(&:attributes)`.

Let's run Cucumber again.

As you can see, now both steps are green, which means our feature is working as expected.

I also want to show you what happens if the database table does not match the one we defined in Gherkin. To do that, let me break this scenario. And let's run Cucumber again.

As you can see, it shows us a nice output exposing the difference in the `first_name` field. It also shows us some extra columns that were actually given to the diff call from the customer attributes we loaded from the database, like `id` and `created_at`, but the columns we don't mention in Gherkin are ignored by default. We could use them if we wanted to, though.

Now, looking back at our feature file, I want you to notice we are using macro steps here. They're succinct and they hide details like filling in HTML inputs, visiting pages, clicking buttons, etc. Those are implementation details.

Now, developers coming from Rspec might write a feature like this.

That's the naive approach and it is **not** recommended. This example is too detailed. Cucumber is about focusing on describing WHAT a feature does and not HOW it does it.

We could now go ahead and define our next feature here, which would make sure we properly handle an invalid submission and show validation errors, but I'm gonna leave that as an exercise for you if you want to give it a go. This repository is linked down below in the show notes, clone it and have fun.

Hopefully, this gives you a very good idea on how Cucumber can help you and how it's supposed to be used. It's not a perfect solution, there are still lower level scenarios I prefer testing with Rspec, like APIs, controller responses and user permissions. Still, it works. I have been using it widely for years in my personal projects and I only write Rspec tests when Cucumber is really not a good fit.

What I showed you here is just the tip of the iceberg. The Gherkin language has a lot more features to offer. You can learn more from the official Gherkin Reference linked below.

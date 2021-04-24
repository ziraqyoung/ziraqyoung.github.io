---
title: "Using Rake to seed you Database"
date: 2021-03-07 12:02:20 +0300
categories: rails testing
tags: rails seeding
---

## :link: What is Rake ?

[Rake](https://ruby.github.io/rake/) is a task runner in Ruby. Such include *running specs*, *exporting/importing data from/to the database* etc.

[Rails](https://rubyonrails.org) uses rake to run such tasks eg: `bin/rails db:migrate`, `bin/rails test` etc 

Checkout this post in [Ruby Guides](https://www.rubyguides.com/2019/02/ruby-rake/) for introduction to rake to learn more

## :link: Why seed via custom rake task?

Rails ships with a file in `db/seeds.rb`, where you define you seed data using your domain model.

To seed you run `bin/rails db:seed` assuming all the required migration are run.

For example:

```ruby
# db/seeds.rb

def seed_users
  puts "seeding users..."
  user_id = 0

  10.times do
    User.create(
      name: "#{Faker::Name.name}-#{user_id}",
      email: "test-#{user_id}@test.com",
      password: '123456',
      password_confirmation: '123456'
    )
    user_id = user_id + 1
  end
end

seed_users
```

This code uses the [Faker library](https://github.com/faker-ruby/faker) for generating dummy data,  put it inside a method `seed_users` and call the method to actually seed



This all goes well and you can go millage out of it.



But needs arises when you want to have *different data in production server* but additional *data for development environment*. This is where rake tasks are handy

## :link: Code

Assumption: Let's assumes our app has *posts*, *categories*, *users* and *private conversations*

Also there is no interface to add categories, i.e they are loaded via **db/seeds.rb**

If we want to seed all above resources, it would be nice to load *categories* in  development and production environment but only load *posts*, *user* and *private conversations* in development environment only. 



In **db/seeds.rb**, we create logic to seed *categories only*

```ruby
# db/seeds.rb

# define a method
def seed_categories
  puts "seeding categories..."

  hobby = %w['Arts', 'Crafts', 'Sports', 'Sciences', 'Collecting', 'Reading', 'Other']
  study = ['Arts and Humanities', 'Physical Science and Engineering', 'Math and 	Logic', 'Computer Science', 'Data Science', 'Economics and Finance', 'Business', 'Social Sciences', 'Language', 'Other']
  team = ['Study', 'Development', 'Arts and Hobby', 'Other']

  hobby.each do |name|
    Category.create!(
      branch: "hobby",
      name: name
    )
  end

  study.each do |name|
    Category.create!(
      branch: "study",
      name: name
    )
  end

  team.each do |name|
    Category.create!(
      branch: "team",
      name: name
    )
  end
end

# call the method
seed_categories
```

To seed *categories*, *posts*, *users*, *private conversations*, we define a rake task

In `lib/tasks` directory, create a rake task using the rails generator

- `bin/rails g task db all`, this will require you to wrap *namespace :seed ..*, as shown in the file below

* create a file name *db_seed_all.rake*. Note the `.rake` at the end of the file

```ruby
# lib/tasks/db_seed_all.rake

namespace :db do
  namespace :seed do
    desc "Seeds all"
    task all: [:environemt, 'db:seed'] do
      seed_users
      seed_posts
      seed_private_conversations
    end
  end
end


# methods
private
	# Seeds users
		def seed_users
  		puts "seeding users..."
  		user_id = 0

    10.times do
      User.create(
        name: "#{Faker::Name.name}-#{user_id}",
        email: "test-#{user_id}@test.com",
        password: '123456',
        password_confirmation: '123456'
      )
      user_id = user_id + 1
    end
  end

  # Seeds posts
  def seed_posts
    puts "seeding posts..."
    categories = Category.all

    categories.each do |category|
      5.times do
        Post.create(
          title: Faker::Lorem.sentences[0],
          content: Faker::Lorem.sentences[0],
          user_id: rand(1..9),
          category_id: category.id
        )
      end
    end
  end

# Seed private conversation
def seed_private_conversations
  puts "Seeding private conversations"
  
  #....
end
```

#### :sparkling_heart: Keypoints:

- Running `bin/rails db"seed`, will only seed categories

- Name-spacing allows running the task with scope, the following task is run via `bin/rails db:seed:all` 

  ```ruby
  namespace :db do
    namespace :seed do
      desc '****'
      task :all
    end
  end
  ```

- The *desc* method allows the task to be visible when you run `rake -T` or `bin/rails`

- Note the use of: `task all: [:environment, 'db:seed']...`. This is to say this task `db:seed:all` is dependent  on another tasks `:environment` and `db:seed` (tasks to be run BEFORE this one) . And that make sense, we do not to seed categories then seed other resources with two commands,  and for `:environment`, rails needs an environment when running tasks

#### :warning: Gotchas!

- Resetting the database with `bin/rails db:reset` wont run this task, it will just seed categories only. Hence do not run it, instead user `bin/rails db:drop`, then `bin/rails db:create` then `bin/rails db:seed:all`. You could define a task that just does that which is awesome. I will leave that part out.

### :tada: Take

If you want data (or seed data) to be present in production environment, then *seeds* in *db/seeds.rb* are an excellent candidate for this, for data need in developemnt using rake task to seed
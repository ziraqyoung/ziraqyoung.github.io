---
title: "Rails Routing Constraints"
date: 2021-04-05 09:00:20 +0300
categories: rails routing
tags: rails routing constraints
---

### When to use Routing Constraints in Rails?

Constraint allows the router to behave differently based on the request at the routing level instead of controller level. eg showing a different homepage for different users or restricting URLs for some sub-domain.



### Practical Example

Lets assume you are building a Rails application with authentication. When users sign up, you want to redirect to home page `home#index`. The you realize you need a home page for signed in users and a different home page for guest user or unauthorized users



#### Solution 1

You could modify to `HomeController` with a `before_action` to redirect to `DashBoardController` if user is signed_in

```ruby
# app/controllers/home_controller.rb
class HomeController < ApplicationController
  before_action :redirect_to_dashboard_if_signed_in
  
  def index
  end
  
  private
  	def redirect_to_dashboard_if_signed_in
      # assume signed_in? commes for clearance ruby gem
      if signed_in?
        redirect_to dashboard_path
      end
    end
end
```

And in the *config/routes.rb* file

```ruby
# config/routes.rb
Rails.application.routes.draw do
  resource :dashboard, only: [:index]
  resource :home, only: [:index]
end
```

This delegate routing decision to a controller. 

But we can do better, using Routing constraints

### Solution 2: Routing Constraints

But before that, lets dig a little bit of routing constraint constraints is



There are several ways to pass constraints to routes

1. :point_right: Segment Restriction

Involves using `:constraints` option eg.

```ruby
# config/routes.rb
match '/:year/:month/:day' => "info#about",
	:constraints => { :year => /\d{4}/, :month => /\d{2}/, :month => 	/\d{2}/ }
```

With this constraint, visiting *localhost:3000/foo/bar* will raise a Routing Error, but *localhost:3000/2020/05* will match the constraint and the request be handled successfully by info controller about action

2. :point_right: Request based constraint

Involves constraining routes based on any method of the `request object`. For example based on *subdomain* or *User-Agent* or really any method. eg

```ruby
# config/routes.rb
match '/:year/:month/:day' => "info#about",
  :constraints => { user_agent: /Firefox/ }
```

This works only if the requesting agent matches matches *Firefox*. Other methods of usable in the `Request object` include `:subdomain`, `:format` etc

3. :point_right: Dynamic Constraint

This uses `matches?` method, either by using a `lambda` or `defining a class that has :matches?` as a `class method` or an `instance method`

- Using `lambda`

```ruby
# config/routes.rb
get '*path',
	to: 'proxy#index',
	constraints: lambda { 
    |request| request.env['SERVER_NAME'].match('foo.bar')
  }
```

Pass the `request` object on the *lamda* as a params & you access all its methods

- Using a `Class` with a `class method` or an `instance method`
  - As an Instance method

```ruby
# config/routes.rb
constraints(Subdomain.new) do
  get '*path', to: "proxy#index"
end

# eg in lib/subdomain.rb
class Subdomain
  def matches?(request)
  (request.subdomain.present? &&
    request.subdomain.start_with?('foobar')
  )
  end
end


```

- - As a Class method

```ruby
# config/routes.rb
constraints Subdomain do
  get '*path', to: 'proxy#index'
end

class Subdomain
  def self.matches?(request)
    (request.subdomain.present? &&
      request.subdomain.start_with?('foobar')
     )
  end
end
```

### Our Example Use Case

We want to have a dashboard for guest users and logged in users. Here are the steps

1. :point_right: Configure *config.routes.rb* to

```ruby
# config/routes.rb

Rails.application.routes.draw do
  constraints SignedInHome.new do
  	root to: "dashboard#show", as: :dashboard
  end
  
  root to: "home#show"
end
```

Note: There are 2 routes for *root*, one within the constraint and one outside. The **order is important** to work.

Also we provide the dashboard home route a name `dashboard_path` and `dashboard_url` using `as: :dashboard` option in the routes declaration.

2. :point_right: Define `SignedInHome` class.

Let's define the class inline but it can be created in the *lib* directory and be imported

```ruby
# config/routes.rb
Rails.application.routes.draw do
  #....
end

class SignedInHome
   def initialize(&block)
   	@block = block || lambda { |user| true }
   end

   def matches?(request)
   	@request = request
     # logic to check if user is authenticated 
     # eg checking user_session is present and authenticated
    user_session.present? && @block.call(current_user)
   end
end
```

This code block is very basic, and assumes the `user_session` is set and the `route` within this constraint can respond to `current_user`



### :link: Resource

- This blog is heavily borrowed from [Thoughbot Clearance gem](https://github.com/thoughtbot/clearance) and specifically [Clearance::Constraints::SignIn module](https://github.com/thoughtbot/clearance/blob/729ed73d66fb64caa48ec2c206018322a4ffc476/lib/clearance/constraints/signed_in.rb)

- Also the Rails Documentation on [Rails Routing From Inside](https://guides.rubyonrails.org/routing.html)
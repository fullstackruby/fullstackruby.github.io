---
layout: post
title: "A Volt Chat Application in Ten Minutes"
date: 2015-05-27 11:31:48 -0700
comments: true
categories: ruby, webdev, volt
---

# Screencast - [@rickcarlino](http://www.github.com/rickcarlino)

<iframe width="500" height="280" src="https://www.youtube.com/embed/rc4GR04KUy0" frameborder="0" allowfullscreen></iframe>

# Asciicast - [@ybur-yug](http://www.github.com/yburyug)

[Source](https://github.com/DataMelon/volt-chat-demo)

<!-- more -->

## Building a Realtime Chat Application Using Only Ruby

## Why?
If you want to build a realtime chat app using Ruby, Volt makes a great choice because all of the updates happen in realtime like you
see here, and you can write your frontend and backend code in Ruby. If you have not heard of Volt, follow along and you will start to
see some great benefits. In this case, we will especially illustrate how to handle persistance and ownership for all of our data.

## Setup

Like always, we're going to create a new app using the `volt new` command

```
volt new chat
```

then we can `cd chat` to get to directory and get started.

## Models!

First of all, we are writing in Ruby. So, Objects are our best friend! In Volt's case, the standard model object is `Volt::Model`.
This will let us encapsulate business logic, pesistance, and relational pieces with other parts of the code in one simple area.
For those of you who may be wondering, Volt is a MVVM framework, not MVC. For the curious, read further [here](http://stackoverflow.com/questions/667781/what-is-the-difference-between-mvc-and-mvvm).

To make the model, 

```
volt g model message
```

`vim app/main/models/message.rb`
I prefer to use `vim`, but replace that keyword with whatever the *nix command for your editor of choice is.

```ruby
class Message < Volt::Model
end
```
As you can see, this is a very simple class, that outside of inheritance, looks like a regular Ruby class.

MongoDB is a schemaless database, so we don't have to define our fields ahead of time. In today's example, we will though. This gives us the ability to add type validations and some other nice features I'll cover later.

We can declare a field's name and type via the `field` method. This makes life easier by keeping our data relatively consistent.

```ruby
class Message < Volt::Model
  field :body, String
end
```

If we use the `field` helper, we get some wonderful validations and sanitazation for our input for essentially free.


## Users!

Since this is a chat app, we're going to need a way of handling ownership. Luckily, Volt comes with user management by default, and also
has some very convenient helpers.

```ruby
class Message < Volt::Model
  own_by_user
  field :body, String
end
```

`own_by_user` adds a `user` field to the model and assigns the `current_user` to that model when it's created in the browser.
It really is this easy. I know it's hard to believe.

It also adds some nice authorization tools to handle permissions, but we won't cover that today. For more information, see the Volt API docs.

## The Controller

Though Volt favors a component-based nature, for the sake of simplicity we will continue to isolate this in our `main` component
and just use it's controller for the sake of brevity. Let's open it and peek at the default.

`vim app/main/controllers/main/main_controller.rb`

```ruby
module Main
  class MainController < Volt::ModelController
    def index
      # Add code for when the index view is loaded
    end

    def about
      # Add code for when the about view is loaded
    end

    private

    # The main template contains a #template binding that shows another
    # template.  This is the path to that template.  It may change based
    # on the params._component, params._controller, and params._action values.
    def main_path
      "#{params._component || 'main'}/#{params._controller || 'main'}/#{params._action || 'index'}"
    end

    # Determine if the current nav component is the active one by looking
    # at the first part of the url against the href attribute.
    def active_tab?
      url.path.split('/')[1] == attrs.href.split('/')[1]
    end
  end
end
```

The `index` action is executed in the browser whenever the user visits the root path of the site, as one might expect.

This is a chat app, so we're going to need users to sign in before letting them chat. To do this, we use a `before_action` handler.


```
    before_action :require_login, only: :index
```

If you've worked with Rails or Devise, this syntax should look pretty familiar.

## Bootstraping a view

Since we are in our main component, let's open its view. 

`vim app/main/views/main/index.html`

```html
<:Title>
  Home

<:Body>
  <h1>Home</h1>
```

We need a menu for users to chat with. Here is what we have.

```html
  ...
  <div class="row">
    <div class="col-lg-12">
      <div class="panel panel-default">
        <div class="panel-heading">Chat</div>
        <div class="panel-body">
          <form class="form-inline center" e-submit="send_message">
            <div class="form-group">
              <input type="text" class="form-control" placeholder="Enter a message" value="{{ page._form._body }}">
            </div>
            <input type="submit" class="btn btn-default"/>
          </form>
          <hr/>
            <div>
              {{ store._messages.each do |message| }}
                <p>
                  <strong>{{ message.user.then(&:_name) }}</strong>
                  <span>{{ message._body }}</span>
                </p>
              {{ end }}
            </div>
        </div>
      </div>
    </div>
  </div>
  ...
```

Let's talk about what's going on here.

You might have noticed that we have some unfamiliar attributes on our tags, such as the `e-submit` attribute you see on this `<form>`
tag. We also have some double mustache templates that contain ruby code. These features are known as "bindings"

## Bindings

Be sure to follow up with [the documentation](http://docs.voltframework.com/en/docs/attribute_bindings.html) for binding as well.

As the name suggests, bindings are a way of binding data to your views. This all happens using Ruby- even on the front end.

Inside the form tag, we were using an `e-submit` binding. That binding allows us to call a controller method whenever a form gets
submitted. We haven't written that controller method yet, but when we do, it will handle saving a message to the database. 
We won't need to worry about publishing the changes to other users, because Volt handles that for us.

## Two Way Binding

```
...
    <input type="text" class="form-control" placeholder="Enter a message" value="{{ page._form._body }}">
...
```

I'd like to point out something on our input tag. This binding right here is a two way data binding for a yet to be defined model. 
When we change the input value, it will change our model and vice versa. You might have also noticed I'm binding to a variable 
called page.

 So what is this page variable doing exactly?

## Provided Collections

[Documentation about provided collections](http://docs.voltframework.com/en/docs/provided_collections.html)

Volt comes with several pre-defined collections. A collection is a group of models with varying levels of persistence, depending 
on the use case.

In our case, we're using the `page` collection, which is just a temporary holding area for models.

Later on, we will use the 'store' collection, which uses MongoDB for persistence.

## More Models
[Documentation](http://docs.voltframework.com/en/docs/models.html)]

You might have noticed that I was prefixing our model attributes with an underscore, as seen here in the documentation.

In Volt, you can think of Models as objects similar to Hashes. They are usually schemaless, so you don't neccessarily need to define
the attributes ahead of time, sort of like how a hash works, but unlike hashes, you access data using underscore attributes instead
of square brackets.

Let's open up a console.

`bundle exec volt c`

To illustrate the use of underscore attributes, let's make some models from the Volt console.

In these examples, we will work with the `Message` model that we defined earlier.

Since our Message model is stored in a schemaless MongoDB collection, that means we can add any attributes we need at runtime by using underscore attributes.

```ruby
message = Message.new
message._anything = "Example"
message
message._anything
```

Since we declared an attribute called 'body' at the begining of this video, we don't need an underscore to access that attribute.

```ruby
message.body
message.body = 'Hello, world!'
message
```

```
...
              <input type="text" class="form-control" placeholder="Enter a message" value="{{ page._form._body }}">
...
```

With our new understanding of how models work, we must populate an empty message model for our form to use. This will need to happen in the controller

Our form is bound to page._form, so I will first write a method that populates some empty data.

`vim app/main/controllers/main/main_controller.rb`

```ruby
    def reset_message
      page._form = Message.new
    end
```

Then I will add that to the index action so that it gets called on page load

```
    def index
      reset_message
    end
```

```
...
          <form class="form-inline center" e-submit="send_message">

...
```

Now to handle form submissions, we need to save the message object to the database and then clear out the form afterwards. As you can see by the `e-submit` binding, this behavior will be defined in a controller method named `send_message`

Back to `main_controller`...

When we define `send_message`

```ruby
...
    def send_message
    end
```

we need to add the form model to `store` collection

```ruby
...
      store._messages << page._form
...
```

then reset the form model so that the text box clears out

```ruby
      reset_message
```
(highlight source code of `reset_message`)

Let's go to our browser, visit `localhost:3000` after running `bundle exec volt s`

It looks like we're done. As you can see, it's asking us to log in.

That's all there is to it. I hope you've enjoyed watching. Feel free to leave comments if you have any questions.

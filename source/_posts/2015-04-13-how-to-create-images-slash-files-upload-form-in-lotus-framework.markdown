---
layout: post
title: "How to create images/files upload form in Lotus Framework?"
date: 2015-04-13 09:25
comments: true
categories:
tags: lotusrb
author: "Trung Lê"
---

{{ post.title }}

In today tutorial, I will show you how to create an image uploader with Lotus.

You will learn:

* Create file input and multipart HTML form
* Process uploaded file request
* Good practices on making your code nice and clean

<!--more-->

## Prerequisites

Lotus is progressing so quickly that I am afraid this post might get out-of-date
quickly. I will try my best to keep it up-to-date as much as I can. For the time
being, please ensure you meet following requirements:

* Lotus 0.3.1

## Initial Setup

Generate a new demo app:

```
lotus new demo
cd demo
```

The above command will generate a new lotus container with one web app that
resides in `apps/web`.

## Create Web Form

### Make new controller actions

I create a new RESTful controller action

```
bundle exec lotus generate action web images#new
```

which would create a new `Web::Controllers::Images` controller where we will
put our form for the image resource.

We also need `images#create` controller to handle submitted form:

```
bundle exec lotus generate action web images#create
```

By default, lotus action generator generates a GET route to `images#create`. We
tweak this route to POST by modifying the `apps/web/config/routes.rb`:

```ruby
get '/images', to: 'images#new'

# from
# get '/images', to: 'images#create'

# to
post '/images', to: 'images#create'
```

Even better, we can turn our route to be RESTful resource by getting rid of
our 2 existing routes and replace them with one line route:


```ruby
resources :images, only: [:new, :create]
```

Let's double-check before moving on by command:

```
$ bundle exec lotus routes
new_images GET, HEAD  /images/new  Web::Controllers::Images::New
    images POST       /images      Web::Controllers::Images::Create
```

The output above indicates that our app now has 2 routes that we wanted :).

Besides, we should also clean up the view part of create action:

```
rm apps/web/templates/images/create.html.erb
rm apps/web/views/images/create.rb
```

### Create the upload form HTML template

We add our HTML form by modifying `apps/web/templates/images/new.html.erb`:

```
<form method="post" action="<%= Web::Routes.url(:images) %>" enctype="multipart/form-data">
  <input type="file" name="image" id="image">
  <input type="submit" value="Upload">
</form>
```

Please pay attention that our `form` uses POST method with action attribute points to
the `images#create` action and encryption type is `multipart/form-data`. The last
option tells Lotus that we are going to take in non-text or binary submitted data.
Additionaly, we have one input field of type file so user could choose file to upload.

Let's give it a check, start the server first with:

```
bundle exec lotus server
```

and browser `http://localhost:2300/images/new` and you see our new form.

## Refactor upload form with lotus helpers (optional)

While we are on this template layer, I'd like to show you how to create a template
helper for this upload form. Unlike Rails, Lotus has a clear separation between
the controller and the presenter. In other words, Lotus's controller does not
render HTML template, but taking parsed request parameters then calling to
model for processing and delgate the presentation to View class (presenter class).

We create our upload form in `Web::Views:Images` class. We modify `apps/web/views/images/new.rb`
file with following content:


```ruby
module Web::Views::Images
  class New
    include Web::View
    include Lotus::Helpers

    def upload_form
      html.form(method: 'post', action: create_images_url, enctype: 'multipart/form-data') do
        input(type: 'file', name: 'image')
        input(type: 'submit', value: 'Upoad')
      end
    end

    private

    def create_images_url
      Web::Routes.url(:images)
    end
  end
end
```

The example codes above indicates the inclusion of `Lotus::Helpers` of which `html`
method that we used in `Web::Views::Images#upload_form`. Lotus makes writing
helpers simple, users don't have to deal with string concatenation like Rails.
And I think you can work out how to use Helpers on yourself.

Please make sure we update our template to use our new presenter method, by
modifying `apps/web/templates/images/new.html.erb` file with:

```
<%= upload_form %>
```

Please be noted that future version will likely introduce `form_for` method
for convenience.


### Handle submitted payload in controller action

The last part of the tutorial is to handle the submitted image in controller.

We modify our `Web::Controllers::Images::Create`:

```ruby
module Web::Controllers::Images
  class Create
    include Web::Action

    def call(params)
      dest_dir = Web::Application.configuration.root.join('public/uploads')
      FileUtils.mkdir_p(dest_dir)
      tempfile = params['image']['tempfile']
      filename = dest_dir.join(params['image']['filename']).to_s
      begin
        FileUtils.cp(tempfile.path, filename)
        self.body = "#{filename} has been successfully uploaded."
      rescue => e
        self.body = "Failed to upload #{filename} due to: #{e.backtrace.join("\n")}"
      ensure
        tempfile.close
        tempfile.unlink
      end
    end
  end
end
```

Let's dive in a bit, as you can see in the above code, `params['image']` has
the uploaded file (which is now temporarily stored) and the filename. We copy this
temp file to to `apps/web/public/uploads`. You can see this directory is always
get created with `mkdir_p` to ensure we always have this folder setup. We could
refactor the code by having this folder created manually by hand and getting rid
of that line. Lastly, it's a good practice to clean up tempfile :)

You can give this a test go by using the browser and try to upload file. If nothing
goes wrong, you should see a new page with message filenmas has been successfully
uploaded.


## Refactor controller action with interactor (optional)

We could refactor our controller code to have less intimate knowledge about the
file moving logic by moving those chunks of business domain logics to Service Object.
Fortunately, Lotus offers [Lotus::Interactor](https://github.com/lotus/utils/blob/master/lib/lotus/interactor.rb) right out of the box for this scenario.

We create a new folder for our interactor:

```
mkdir apps/web/interactors
```

And tell `Web::Application` to load this path by modifying `apps/web/application.rb`:

```ruby
load_paths << [
  'controllers',
  'views',
  'interactors' # <= we add this new line
]
```

then create our service class at `apps/web/interactors/upload_image.rb`:

```ruby
require 'lotus/interactor'

class UploadImage
  include Lotus::Interactor

  attr_reader :tempfile, :filename

  def initialize(tempfile, filename)
    dest_dir = get_upload_path
    @tempfile = tempfile
    @filename = dest_dir.join(filename)
  end

  def call
    begin
      FileUtils.cp(tempfile.path, filename.to_s)
    rescue => e
      e.backtrace.each { |msg| error msg }
    ensure
      tempfile.close
      tempfile.unlink
    end
  end

  private

  def root
    Web::Application.configuration.root
  end

  def get_upload_path
    dir = root.join('public/uploads')
    FileUtils.mkdir_p(dir)
    dir
  end
end
```

and change our controller to:

```ruby
module Web::Controllers::Images
  class Create
    include Web::Action

    def call(params)
      tempfile = params['image']['tempfile']
      filename = params['image']['filename']

      result = UploadImage.new(tempfile, filename).call

      if result.success?
        self.body = "#{result.filename} has been successfully uploaded."
      else
        self.body = "Failed to upload #{filename} due to: #{result.errors.join("\n")}"
      end
    end
  end
end
```

Interactor returns a result object after calling `#call`. The result object
will determine the success of the usecase by checking if there is any errors.
If there are any, we tell controller to list them out with result errors.

As you can see, controller now only do params parsing and delegate the responsiblity
to the use case object. This is a beautiful patter that I think Rails should
have by default.

If you pay close attention, you can see that we do not have any codes to
check for the presence of required params. If you are familiar with Rails, you
might think of something similar to Strong Parameters for whitelisting params
and having Service class check for the presence. With Lotus, you could easily
do that within your controller:

```ruby
module Web::Controllers::Images
  class Create
    include Web::Action

    params do
      param :image do
        param :tempfile, presence: true
        param :filename, presence: true
      end
    end

    def call(params)
      tempfile = params['image']['tempfile']
      filename = params['image']['filename']

      result = UploadImage.new(tempfile, filename).call

      if result.success?
        self.body = "#{result.filename} has been successfully uploaded."
      else
        self.body = "Failed to upload #{filename} due to: #{result.errors.join("\n")}"
      end
    end
  end
end
```

As you can see in the above code, we added new `params` block. We tell Lotus to whitelist
the nested params `:image` as well as check for presence of `:tempfile` and `:filename`.

## Conclusion

I hope you are now familiar with the process uploading file in Lotus. Here is
few key points that you should be remember:

* Create upload HTML form with POST type and `enctype="multipart/form-data"`
* Controller access to upload files via `params['image']`
* Business Logic can be extracted to Interactor class
* Params whitelisting/validation can be done within controller action

If you have any question, please do not hesitate to ask.

The sample code can be found at [github.com/ruby-journal/lotus-file-upload-demo](https://github.com/ruby-journal/lotus-file-upload-demo)

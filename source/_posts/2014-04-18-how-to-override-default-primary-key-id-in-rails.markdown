---
layout: post
title: "How to override default primary key id in Rails"
date: 2014-04-18 10:28
comments: true
tags: ruby rails
author: "Trung Lê"
---

{{ post.title }}

Rails is all about Convention over Configuration, this includes the DB primary key,
which is always set to be `id` column. What if you want to use different column
as your primary key? Read on and I'll show you how.

<!--more-->

NOTE: This article is written for Rails 4.1.1 /w Postgres

## Usecase

My webstore uses a special string ID, called `sku`, this ID is unique and queried
instead of `id` column. For the sake of simplicty, I use Rails scaffolding:

```
$ rails g scaffold products sku
```

## What is primary key?

A column is called a primay key when it is:

* has not NULL constraint
* has UNIQUE contraint
* has index

We could prove this by checking out DB schema:

```
$ rails db
psql (9.3.4)
Type "help" for help.

webstote_development=# \d products
                                     Table "public.products"
   Column   |            Type             |                       Modifiers
------------+-----------------------------+-------------------------------------------------------
 id         | integer                     | not null default nextval('products_id_seq'::regclass)
 sku        | character varying(255)      |
 created_at | timestamp without time zone |
 updated_at | timestamp without time zone |
Indexes:
    "products_pkey" PRIMARY KEY, btree (id)
```

The above output clearly indicates that at DB level, is is NOT NULL, is PRIMARY_KEY and indexed.

According to Postgres doc, PRIMARY_KEY constraint is a combination of a unique constraint and a not-null constraint.

## Migration

We have following migration:

```ruby
class CreateProducts < ActiveRecord::Migration
  def change
    create_table :products do |t|
      t.string :sku

      t.timestamps
    end
  end
end
```

and we want to turn the `sku` column to be our primary key.

By convention, you would not see any code mention about the primary `id` column.
This column is automatically created when you call method `create_table`.

We can tell Rails not to create column id by parsing `id: false` options

```ruby
  ...
  create_table :products, id: false do |t|
    ...
  end
  ...
```

and then we tell Rails to turn `sku` to be primary key. After doing abit of research,
I found method `ActiveRecord::ConnectionAdapters::TableDefinitiion#primary_key` with
following code:

```ruby
# File activerecord/lib/active_record/connection_adapters/abstract/schema_definitions.rb, line 68
def primary_key(name, type = :primary_key, options = {})
  column(name, type, options.merge(:primary_key => true))
end
```

Disecting the above shows that by using type `primary_key`, I could turn my column to primary key.
I wonder what this type is. Let's give it a go:

```ruby
create_table :products, id: false do |t|
  t.primary_key :sku

  t.timestamps
end
```

after `rake db:migrate`, I check my DB schema again:

```
                                      Table "public.products"
   Column   |            Type             |                       Modifiers
------------+-----------------------------+--------------------------------------------------------
 sku        | integer                     | not null default nextval('products_sku_seq'::regclass)
 created_at | timestamp without time zone |
 updated_at | timestamp without time zone |
Indexes:
    "products_pkey" PRIMARY KEY, btree (sku)
```

The `sku` column is assigned with `integer` datatype, which is not correct. So `primary_key` type must be of `integer` type.

Naively, I try other alternatives:

```ruby
create_table :products, id: false do |t|
  t.string :sku, primary: true

  t.timestamps
end
```

OR

```ruby
create_table :products, id: false do |t|
  t.string :sku, primary_key: true

  t.timestamps
end
```

which doesn't do anything. Meh :(

It is okay, if Rails doesn't let you do so, because you can still be able to do a workaround:

```ruby
class CreateProducts < ActiveRecord::Migration
  def change
    create_table :products, id: false do |t|
      t.string :sku, null: false

      t.timestamps
    end

    execute %Q{ ALTER TABLE "products" ADD PRIMARY KEY ("sku"); }
  end
end
```

and bravo, it does the job! Hold on, there is a twist!

our `db/schema.rb` says nothing about the contraint addition line, which makes
sense because we are not using SQL format to preserve it.

```
config.active_record.schema_format = :sql
```

I don't like this solution because I love readability of helper methods than SQL.

I come up with better solution (IMHO), according to Postgres documentation:

"a primary key constraint is simply a combination of a unique constraint and a not-null constraint"

Let's see how I do it:

```ruby
class CreateProducts < ActiveRecord::Migration
  def change
    create_table :products, id: false do |t|
      t.string :sku, null: false

      t.timestamps
    end

    add_index :products, :sku, unique: true
  end
end
```

and let's check our schema:

```
               Table "public.products"
   Column   |            Type             | Modifiers
------------+-----------------------------+-----------
 sku        | character varying(255)      | not null
 created_at | timestamp without time zone |
 updated_at | timestamp without time zone |
Indexes:
    "index_products_on_sku" UNIQUE, btree (sku)
```

Though our sku doesn't have PRIMARY_KEY contrsaint, yet it is equivalent.

And look at our `db/schema.rb`:

```ruby
ActiveRecord::Schema.define(version: 20140510012730) do

  # These are extensions that must be enabled in order to support this database
  enable_extension "plpgsql"

  create_table "products", id: false, force: true do |t|
    t.string   "sku",        null: false
    t.datetime "created_at"
    t.datetime "updated_at"
  end

  add_index "products", ["sku"], name: "index_products_on_sku", unique: true, using: :btree

end
```

All lovely and readable ;)

## Model

We tell our model `Product` to use `sku` as primay key:

```ruby
class Product < ActiveRecord::Base
  self.primay_key = 'sku'
end
```

And you are set to go.

Now I wonder if changing to different primary key, would it cause any side effect
to method like `#find`. The answer is not at all, if we look into `ActiveRecord::AttributeMethods::PrimaryKey`:

```ruby
# File activerecord/lib/active_record/attribute_methods/primary_key.rb, line 17
def id
  sync_with_transaction_state
  read_attribute(self.class.primary_key)
end
```

the `read_attribute(self.class.primary_key)` line tells us that it will reference to
our `Product.primary_key`, that is `sku`. Therefore, we you call:

```ruby
Product.find('SKU-01')
```

it would generate SQL:

```sql
SELECT  "products".* FROM "products"  WHERE "products"."sku" = $1 LIMIT 1  [["sku", 'SKU-01']]
```

and FYI, there is also writer method `#id=` that is provided by Rails:

```ruby
p = Product.new
p.id = 'SKU-02'
p.sku
# SKU-02
```

If you don't like to use `#id` and `#id=`, you could override them in your class:

```ruby
class Product < ActiveRecord::Base
  ....
  def id
    raise NoMethodError, "Please call #{self.class.primay_key} instead."
  end

  def id=(value)
    raise NoMethodError, "Please call #{self.class.primay_key}= instead."
  end

  ....
end
```

Be aware that `Product.find` won't work anymore, and other Rails helper that relies
on `id` will stop functioning. If you really want that, you need to override more
methods and this seems too much of a pain for me. So I'd highly recommend you to
leave `#id` as is.

# Routing

Our default rails is:

```ruby
# config/routes.rb
resources :products
```

which by default will generate routes with reference to `:id`:

```text
bundle exec rake routes
            Prefix Verb  URI Pattern                       Controller#Action
           products GET   /products(.:format)                 products#index
                   POST   /products(.:format)                 products#create
        new_product GET   /products/new(.:format)             products#new
       edit_product GET   /products/:id/edit(.:format)        products#edit
            product GET   /products/:id(.:format)             products#show
                  PATCH   /products/:id(.:format)             products#update
                    PUT   /products/:id(.:format)             products#update
                 DELETE   /products/:id(.:format)             products#destroy
```

With Rails 4, you can change `:id` to `:sku` with:

```ruby
# config/routes.rb
resources :products, param: :sku
```

our routes would be:

```text
bundle exec rake routes
            Prefix Verb  URI Pattern                       Controller#Action
           products GET   /products(.:format)              products#index
                   POST   /products(.:format)              products#create
        new_product GET   /products/new(.:format)          products#new
       edit_product GET   /products/:sku/edit(.:format)    products#edit
            product GET   /products/:sku(.:format)         products#show
                  PATCH   /products/:sku(.:format)         products#update
                    PUT   /products/:sku(.:format)         products#update
                 DELETE   /products/:sku(.:format)         products#destroy
```

Please be noted that you need to modify the `ProductssController` to find Order record
with `params[:sku]` instead of `params[:id]`.

```ruby
# app/controllers/products_controller.rb
def set_product
  @product = Product.find(params[:sku])
end
```

## URI

For example if we have a Product with a very non-URI friendly SKU like: 'SKU 001',
when we access our `ProductsController#show`, we would have following URL:

```
http://0.0.0.0:3000/products/SKU%23001
```

to make our primary key URI friendly, we could override `#to_param` method. By
convention, Rails would call this method to work out the routing URI:

```ruby
class Product < ActiveRecord::Base
  ...
  def to_param
    sku.parameterize
  end
  ...
end
```

and the URI turns to:

```
http://0.0.0.0:3000/products/SKU-001
```

## Conclusion

Rails is all about CoC, for most of the cases, you should stick to it. However you could
override the convention by:

* Adding NOT NULL and UNIQUE contrainst to column you want to be the primary key
* Set `self.primary_key` for the model
* Set `param` for your routing
* Override `#to_param` for friendly URI

That's all for today. Please leave comments if you have any question.
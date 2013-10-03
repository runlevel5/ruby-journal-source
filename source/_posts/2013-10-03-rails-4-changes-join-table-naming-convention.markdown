---
layout: post
title: "Rails 4 changes habtm join table naming convention"
date: 2013-10-03 18:55
comments: true
tags: rails activerecord
author: "Trung Lê"
---

{{ post.title }}

When upgrading my app from Rails 3.2.14 to Rails 4, I bumped into an error in which
habtm association complains missing DB join table. To my surprise, Rails 4 has changed the default naming convention for join table.

<!--more-->

Here is the Rails 3 code:

```ruby
class ContactPlan < ActiveRecord::Base
  has_and_belongs_to_many :contact_types
end


class ContactType < ActiveRecord::Base
  has_and_belongs_to_many :contact_plans
end

class CreateContactPlansContactTypes < ActiveRecord::Migration
  create_table :contact_plans_contact_types, id: false do |t|
    t.integer :contact_plan_id, null: false
    t.integer :contact_type_id, null: false
  end
end
```

As you can see from above migrarion, my join table naming is `contact_plans_contact_types`. Yet Rails 4 fails to run the migration and asks for the existence of `contact_plans_types`. If you are upgrading from Rails 3 app, I highly recommend you avoid renaming this join table. Instead, you should explicitly declare the join table for habtm association:

```ruby
class ContactPlan < ActiveRecord::Base
  has_and_belongs_to_many :contact_types, join_table: :contact_plans_contact_types
end

class ContactType < ActiveRecord::Base
  has_and_belongs_to_many :contact_plans, join_table: :contact_plans_contact_types
end
```

This technique mitigates regressions. And you can always go back to renaming if you feel like after stablising the build.

I can see that Rails 4 try to be smart by detecting duplication in model naming. I welcome this sort of change as it would reduce the complexity in the naming. Furthermore, if you bump into this issue, it is very likely that you should rethink your model naming.



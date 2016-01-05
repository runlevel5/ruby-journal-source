---
layout: post
title: "Liquid template in Ruby done right"
date: 2014-04-10 16:48
comments: true
tags: liquid ruby
author: "Trung Lê"
---

{{ post.title }}

Liquid Templating Engine is an awesome technology and with the power of gem 'liquid',
everybody can start using without much hassles. I saw many projects used this gem
but sadly most of them are quite bad. In this tutorial, I'll go through a bad example
and show you how to refactor it.

<!--more-->

## The smell

Below code is for a report generation function for a car dealer Rails app.

Let's see the codes:

{% raw %}
```ruby
# app/models/car_issue.rb
class CarIssue < ActiveRecord::Base
  belongs_to :car

  liquid_methods :description, :reference_number
end

# app/models/car.rb
class Car < ActiveRecord::Base
  has_many :car_issues
  belongs_to :customer

  liquid_methods :car_make, :owner, :dealer_contact_details

  def owner
    customer.full_name
  end

  def car_make
    "#{brand} #{model} MY #{manufacturer_year}"
  end

  def dealer_contact_details
<<-CONTACT
  Uber Dealer

  9 Sesame St
  Emo Town
CONTACT
  end

end

# app/models/customer.rb
class Customer < ActiveRecord::Base
  liquid_methods :full_name

  def full_name
    "#{first_name} #{last_name}"
  end
end

# app/services/car_report_generation.rb
class CarReportGeneration
  def initialize(car)
    @car = car
  end

  def template
<<-TEMPLATE
  Dear {{ car.owner }},

  We've performed inspection on your {{ car.car_make }} and found following issues:

  {% for issue in car.issues %}
    * {{ issue.reference_number }} - {{ issue.description }}
  {% endfor %}

  Please call us back for quotes

  Sincerely yours

  {{ car.dealer_contact_details }}
TEMPLATE
  end

  def generate_report
    Liquid::Template.parse(template).render 'car' => @car
  end

end
```
{% endraw %}

As you can see above, the main logic is `CarReportGeneration#generate_report` which
render the report template using Liquid to populate fields.

What's so stink about the above code? In fact, I think it's perfectly fine. However should
the report requires more details, user will create more methods within Car model to serve
as report details getter and this could get very ugly quickly as the coupling emerges more
clearly.

## Refactoring

We will take many little steps.

The first step is to reduce the coupling between our models and Liquid. As you can see
above that the code exposes methods to Liquid with method `liquid_methods`. This method
meta-programmingly create for you a class which is a subclass of Liquid::Drop then create
instance methods that matches the parsed method names.

So our Car's `liquid_methods :car_make, :owner, :dealer_contact_details` would do following implicit things:

```ruby
Car.class_eval do
  def to_liquid
    Car::LiquidDropClass.new(self)
  end
end

class Car::LiquidDropClass < Liquid::Drop
  def initalize(object)
    @object = object
  end

  def owner
    @object.customer.full_name
  end

  def car_make
    @object.car_make
  end

  def dealer_contact_details
    @object.dealer_contact_details
  end

end
```

We see that the magic lies in method `to_liquid` which point to a `Liquid::Drop` class.
This special object is what Liquid template takes in to render the template.

Now we could replicate the logic easily. See the code below:

```
# app/models/car.rb
class Car < ActiveRecord::Base
  has_many :car_issues
  belongs_to :customer

  def to_liquid
    CarMergeField.new(self)
  end

end

# app/merge_fields/car_merge_field.rb
class CarMergeField < Liquid::Drop
  attr_reader :car

  def initalize(car)
    @car = car
  end

  def owner
    car.customer.full_name
  end

  def car_make
    "#{car.brand} #{car.model} MY #{car.manufacturer_year}"
  end

  def dealer_contact_details
<<-CONTACT
  Uber Dealer

  9 Sesame St
  Emo Town
CONTACT
  end

end
```

By moving all Liquid-related methods to a separate class, we leave Car model
clean and slim, well with a cost of one function, that is `Car#to_liquid`, but
I guess we could live with that for now.

But that's the end of refactoring yet, we could see that `CarMergeField#dealer_contact_details`
should not belong to Car model and would be shared between many reports in the future.

So we create a new class for that:

```ruby
class DealerMergeField < Liquid::Drop

  def contact_details
<<-CONTACT
  Uber Dealer

  9 Sesame St
  Emo Town
CONTACT
  end

end
```

and make sure we also remove `CarMergeField#dealer_contact_details` and update our template:

{% raw %}
```ruby
# app/services/car_report_generation.rb
class CarReportGeneration do

  # ...

  def template
<<-TEMPLATE
  Dear {{ car.owner }},

  We've performed inspection on your {{ car.car_make }} and found following issues:

  {% for issue in car.issues %}
    * {{ issue.reference_number }} - {{ issue.description }}
  {% endfor %}

  Please call us back for quotes

  Sincerely yours

  {{ dealer.contact_details }}
TEMPLATE
  end

  def generate_report
    liquid_drops = {
      'car' => @car,
      'dealer' => DealerMergeField.new
    }
    Liquid::Template.parse(template).render(liquid_drops)
  end

end
```
{% endraw %}

Please pay attention closely to the `CarReportGeneration#generate_report`, you could
see that I parse in new liquid drop `dealer` which is an object of `DealerMergeField`.
FYI, the liquid does not have to tie to a model. We now could carry on and apply
the same technique for model CarIssue.

Yet, I am not satisfied, we could push abit further by extraction common liquid
rendering logic into its own class and encourage reusability of this class for
other reports.

{% raw %}
```ruby
# app/services/report_generation.rb
class ReportGeneration
  attr_reader :report

  def initialize(report)
    @report = report
  end

  def generate_report
    Liquid::Template.parse(report.template).render(report.liquid_drops)
  end

end

# app/reports/car_report.rb
class CarReport

  def initialize(car)
    @car = car
  end

  def template
<<-TEMPLATE
  Dear {{ car.owner }},

  We've performed inspection on your {{ car.car_make }} and found following issues:

  {% for issue in car.issues %}
    * {{ issue.reference_number }} - {{ issue.description }}
  {% endfor %}

  Please call us back for quotes

  Sincerely yours

  {{ dealer.contact_details }}
TEMPLATE
  end

  def liquid_drops
    {
      'car' => @car,
      'dealer' => DealerMergeField.new
    }
  end

end

```
{% endraw %}

and to use it we do:

```ruby
car_report = CarReport.new(@car)
ReportGeneration.new(car_report).generate_report
```

## Conclusion

Never let a gem manipulate you. If you see a gem makes you do bad things,
then you should dig deeper. Even writing new thing your own is not a bad
solution. If you don't have time for that, make sure you write abstract
method that could help you untangle the coupling later.

I hope this is useful to some. I really welcome feedbacks.

Keep on learning folks!

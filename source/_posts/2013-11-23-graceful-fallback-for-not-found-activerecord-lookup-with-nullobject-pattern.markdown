---
layout: post
title: "Graceful fallback for not found ActiveRecord lookup with NullObject pattern"
date: 2013-11-23 23:27
comments: true
categories:
tags: rails activerecord pattern
author: "Trung Lê"
---

{{ post.title }}

In my application, I bump to Exception when trying to delegate a method to an unfound ActiveRecord instance. This poses two issues for me:

* Hard to write test for you have to set up fixture/factory correctly
* Not a good user experience to see error on production

I tackle this with NullObject pattern to provide a graceful fallback.

<!--more-->

Firstly, let me show you my code:

```ruby

class Employee < ActiveRecord::Base
  # in schema, each employee has a unique email column as :string type

  def employee_of_the_month?
    # yeah, code to determine
  end
end

class EmployeeLookupService

  def self.find_employee_by_email(email)
    Employee.where(email: email).first
  end

end

class ReportService

  def self.generate_for_email(email)
    employee = EmployeeLookupService.find_employee_by_email(email)

    if employee.employee_of_the_month?
      # do something
    end
  end

end

```

I have a Report which generates report for an employee with matching email. As you can in the above
code, if the employee could be be found by email, `EmployeeLookupService.find_employee_by_email` would yield a `nil` and calling `employee_of_the_month?` on `nil` would raise exception. So how could we make sure that our code could handle this exception gracefully?

## Bad way

Well, there is one quick bad way, that is to add a addtional presence check for employee, so:

```ruby
class ReportService

  def self.generate_for_email(email)
    employee = EmployeeLookupService.find_employee_by_email(email)

    if employee && employee.employee_of_the_month?
      # do something
    end
  end

end
```

As you can see, I add `if employee` clause to ensure the presence of employee. What's wrong with this way? I find many Rails developers code this way, but to me it is not good enough. In term of OO design, I expect `EmployeeLookupService.find_employee_by_email` returns me an object which responds to `employee_of_the_month?` consistently instead of returning me a `nil`. Furthermore, it is not the responsibiltiy of `ReportService` to do presence checkup.

## Good way

Here is how I refactor the code, I use NullObject pattern to create a new class, called `NullEmployee` and ensure that this class has the same interface as `Employee`:


```ruby

class NullEmployee

  def initialize(email)
    @email = email
  end

  def employee_of_the_month?
    nil
  end
```

and I also refactor my lookup code:

```ruby
class EmployeeLookupService

  def self.find_employee_by_email(email)
    Employee.where(email: email).first || NullEmployee.new(email)
  end

end
```

Let's digest what I did above, I make `EmployeeLookupService.find_employee_by_email` to create a new instance of `NullEmployee` class if not found and this `NullEmployee#employee_of_the_month?` always return `nil`. Now we do have a consistency in returned employee object. And testing it would be much more pleasant, we do not have to care about setting up this employee fixture correctly, we could simply stub `employee_of_the_month?` which makes testing faster and reduces coupling.

Now, there is one caveat with this NullObject patter and ActiveRecord, what if we call an ActiveRecord API on this NullEmployee instance. Well, we could add all ActiveRecord API into our class and return nil, right? No, I must be kidding in saying that. There are hundreds of them, our class would look messy. The elegant solution is provide graceful fallback for missing methods by implement `method_missing` call. Here is the code:


```ruby
class NullEmployee

  # whatever there before

  def method_missing(name)
    nil
  end

end
```

Now calling a missing method won't yield any exception.

That's it folks. I hope you enjoy it and please never keep learning and remembering these thumb rules:

* Single Reponsibility
* Interface Consistency
* Never mix persisence layer with business logic layer
* If hard to write test, your code might be wrong

See you in the next article. Peace!

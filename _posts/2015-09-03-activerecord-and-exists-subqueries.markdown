---
layout: post
title:  "ActiveRecord and EXISTS() subqueries"
date:   2015-09-03 04:12:13
categories: rails
---

Real world example: We have models `Order` and `Payment`.
Orders have many payments, so client could try another form of payment if first
try failed for whatever reason.

{% highlight ruby %}
class Order < ActiveRecord::Base
  has_many :payments
end

class Payment
  belongs_to :order
  scope :paid, -> { where(state: %W[authorized charged]) }
end
{% endhighlight %}

problem: How to fetch only "paid" `orders`? In this case, orders having at least one payment in `'authorized'` or `'charged'` state.

First, I've tried to count such orders using activerecord.

{% highlight ruby %}
Order.joins(:payments).where(payments: {state: %W[authorized charged]})
  .count("distinct orders")
{% endhighlight %}

It works, but for some reason resulting join is quite inefficient. And it doesn't help to actually fetch orders. I don't need payments though, I just want to know if corresponding paid payments `exist`!

Let's try plain old SQL (almost):

{% highlight ruby %}
class Order < ActiveRecord::Base
  scope :paid, -> {
    where(
      "EXISTS( SELECT 1 FROM payments WHERE payments.order_id = orders.id" \
      " AND payments.state IN (?))", %W[authorized charged]
    )
  }
end
{% endhighlight %}

Works much faster! But it isn't very DRY. I had to repeat Payment.state condition. Maybe Arel could help me?

{% highlight ruby %}
class Order < ActiveRecord::Base
  scope :paid, -> {
    where( Payment.paid.where("payments.order_id = orders.id").exists )
  }
end
{% endhighlight %}

Turns out, Arel provides `exists` method on all your active record scopes, which returns Arel "node" for `EXISTS(subquery for your scope)`. I just stumbled upon it on Stack Overflow, but couldn't find any mention
of that method in Rails Guides or docs.

You can write `"payments.order_id = orders.id"` portion using Arel too, but I don't find it particularly readable:

{% highlight ruby %}
  ...where(Payment.arel_table[:order_id].eq(Order.arel_table[:id]).exists
{% endhighlight %}

But wait! Isn't `EXISTS()` just another form for `IN ()`? Active record already provides us with convenient
shortcut for `IN (?)` predicates:

{% highlight ruby %}
class Order < ActiveRecord::Base
  scope :paid, -> { where( id: Payment.paid.select(:order_id) ) }
end
{% endhighlight %}

Neat! Check out `Order.paid.to_sql`:

{% highlight sql %}
SELECT "orders".* FROM "orders" WHERE "orders"."id" IN (
  SELECT "payments"."order_id"
  FROM "payments"
  WHERE "payments"."state" IN ('charged', 'authorized')
)
{% endhighlight %}

To my (not so much) surprise, `Order.paid.explain` shows the same query plan (on postgresql, which probably matters there) as previous variant. You're welcome.

So, what's going on there? One more useful but undocumented feature in active record is that you can put ActiveRecord::Relation
instance as argument for `where(column: ...)`, and it will be inserted into resulting query as `IN(subquery)`.
Resulting in one query, not two, as one could expect, and avoiding returning to your application (possibly
huge) subquery result.

P.S. Check out [ActiveRecord::PredicateBuilder.register_handler](http://apidock.com/rails/v4.2.1/ActiveRecord/PredicateBuilder/register_handler/class)! Looks like it could be quite useful with some domain specific value objects. Like `Money`, for example. I'll investigate [further](https://gist.github.com/codesnik/2ebba1940c05b08b17f9).

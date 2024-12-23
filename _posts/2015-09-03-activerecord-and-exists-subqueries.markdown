---
layout: post
title:  "ActiveRecord and EXISTS() subqueries"
date:   2015-09-03 04:12:13
categories: rails
---

(updated to reflect changes in Rails 7+)

Real world example: We have models `Order` and `Payment`.
Orders have many payments, so client could try another form of payment if first
try failed for whatever reason.

{% highlight ruby %}
class Order < ApplicationRecord
  has_many :payments
end

class Payment < ApplicationRecord
  belongs_to :order
  scope :paid, -> { where(state: %W[authorized charged]) }
end
{% endhighlight %}

problem: How to fetch only "paid" orders? In this case, orders having at least one payment in `'authorized'` or `'charged'` state.

First, I've tried to count such orders using ActiveRecord.

{% highlight ruby %}
Order.joins(:payments).merge(Payment.paid).distinct.count
# Order Count (0.3ms)  SELECT COUNT(DISTINCT "orders"."id") FROM "orders"
#   INNER JOIN "payments" ON "payments"."order_id" = "orders"."id"
#   WHERE "payments"."state" IN (?, ?)
#   [["state", "authorized"], ["state", "charged"]]
{% endhighlight %}

It works, but for some reason resulting join is quite inefficient. I don't need payments
though, I just want to know if corresponding paid payments `exist`!

Let's try plain old SQL (almost):

{% highlight ruby %}
class Order < ApplicationRecord
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
class Order < ApplicationRecord
  scope :paid, -> {
    where( Payment.paid.where("payments.order_id = orders.id").arel.exists )
  }
  # and here's how it could be properly inverted
  scope :unpaid, -> {
    where.not( Payment.paid.where("payments.order_id = orders.id").arel.exists )
  }
end

Order.paid.to_a
# Order Load (0.6ms)  SELECT "orders".* FROM "orders" WHERE EXISTS (
#   SELECT "payments".* FROM "payments"
#   WHERE "payments"."state" IN (?, ?) AND (payments.order_id = orders.id)
# ) [["state", "authorized"], ["state", "charged"]]
{% endhighlight %}

Turns out, Arel provides `exists` method on relations, which returns Arel "node" for `EXISTS(subquery for your scope)`.

You can write `"payments.order_id = orders.id"` portion using Arel too, but I don't find it particularly readable:

{% highlight ruby %}
  scope :paid, -> {
    where(
      Payment.paid.where(
        Payment.arel_table[:order_id].eq(table[:id])
      ).arel.exists
    )
  }
{% endhighlight %}

But wait! Could we do without `EXISTS()`, using `IN ()`, for example? Active record already provides us with convenient
shortcut for `IN (?)` predicates:

{% highlight ruby %}
class Order < ApplicationRecord
  scope :paid, -> { where( id: Payment.paid.select(:order_id) ) }
end
{% endhighlight %}

Check out `Order.paid.to_sql`:

{% highlight sql %}
SELECT "orders".* FROM "orders" WHERE "orders"."id" IN (
  SELECT "payments"."order_id"
  FROM "payments"
  WHERE "payments"."state" IN ('charged', 'authorized')
)
{% endhighlight %}

It's a subselect, not a list of values. Neat!

To my (not so much) surprise, `Order.paid.explain` shows the same query plan (on postgresql, which probably matters there) as variant with `EXISTS`. You're welcome.

So, what's going on there? One more useful but undocumented feature in active record is that you can put ActiveRecord::Relation
instance as argument for `where(column: ...)`, and it will be inserted into resulting query as `IN(subquery)`.
Resulting in one query, not two, as one could expect, and avoiding returning to your application (possibly
huge) subquery result.

One more approach for creating inverted :unpaid scope exists since Rails 6.1, using LEFT OUTER JOIN:

{% highlight ruby %}
class Order < ApplicationRecord
  has_many :payments
  has_many :paid_payments, -> { paid }, class_name: "Payment"
  scope :paid, -> { where.associated(:paid_payments).distinct }
  scope :unpaid, -> { where.missing(:paid_payments) }
end

# Order.paid.to_sql
# => SELECT DISTINCT "orders".* FROM "orders"
#      INNER JOIN "payments" "paid_payments"
#      ON "paid_payments"."state" IN ('authorized', 'charged')
#         AND "paid_payments"."order_id" = "orders"."id"
#    WHERE "paid_payments"."id" IS NOT NULL

# Order.unpaid.to_sql
# => SELECT "orders".* FROM "orders"
#      LEFT OUTER JOIN "payments" "paid_payments"
#      ON "paid_payments"."state" IN ('authorized', 'charged')
#         AND "paid_payments"."order_id" = "orders"."id"
#    WHERE "paid_payments"."id" IS NULL
{% endhighlight %}

I'm not sure why `paid` scope has `IS NOT NULL` with `INNER JOIN` though, seems superfluous.

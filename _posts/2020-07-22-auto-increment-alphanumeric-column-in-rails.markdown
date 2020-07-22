---
layout: post
title:  "Auto increment alphanumeric column in rails."
date:   2020-07-22
---

A blog post about rails in 2020.  I still use rails as a backend json api and recently implemented something that I think is worth writing up.

I have been migrating some applications from integer primary keys to uuid primary keys.  Universally unique identifiers are not great for display purposes. No one wants to receive an invoice with a 32 hexadecimal character id.

The solution to this problem is pretty trivial. Add a column for vanity identification.

In these applications I had already implemented such a scheme as they were multi tenanted.  So company A and company B could both have an invoice with the `display_id` of say `1`.

However there was a new requirement. The `display_id` needed to be alphanumeric and auto increment. By utilizing the sql `CAST` function I was able to select records with a numeric `display_id` and use that value to auto increment.

```ruby
#app/models/concerns/display_id_concern.rb
module DisplayIdConcern
  extend ActiveSupport::Concern

  included do
    before_validation :generate_display_id

    validates :display_id,
      presence: true,
      uniqueness: {
        scope: [
          :deleted_at
        ]
      },
      format: {
        with: /[\w\-]+/
      }
  end

  private

  def generate_display_id
    self.display_id = next_display_id if display_id.blank?
  end

  def next_display_id
    order_statement = Arel.sql("CAST(display_id AS UNSIGNED) DESC")
    record =
      self.class.base_class.
      where(deleted_at: nil).
      order(order_statement).
      limit(1).
      first

    id = record ? record.display_id.to_i : 0;

    return id + 1
  end
end

```

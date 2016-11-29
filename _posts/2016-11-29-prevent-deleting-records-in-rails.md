---
layout: post
title: "Prevent deleting records in Rails"
date: 2016-11-29
---

Obviously, deleting records from a database is not a feature that most developers would like to incorporate into their application easily. Keeping a record, rather than simply deleting usually makes us stay on the safe side - it is always better to have an old, perhaps redundant, information, at least for future debugging purposes. Even on the attributes level, you might think that keeping the good ol' `created_at` and `update_at` fields in your model isn't that vital - not until at some point in time you want to figure out when a problematic record was created/updated. 

Coming back to Rails and, especially, ActiveRecord, let's think about how we could prevent the deletion of a database record.

1. Delete it anyway... kind of

Perhaps one of the most popular solutions to this problem would be not to directly prevent the record from being deleted. Instead, after calling `delete` or `destroy` the record would be kept in the database without raising an exception, and will only be removed from the default scope via a special `deleted` flag set to true etc. There are several gems providing this functionality - you may check out the paranoia or soft_deletion gems. Of course, there is a way to actually delete the record "physicaly" - paranoia requires us to use the `really_delete` method - whose name suggest we should take a moment to make sure the record needs to go to the bin for sure. !! links go here !!

2. Please don't do it.

As for now, there is no 'standard' way of preventing record deletion, thus we must relate on gems or our own simple implementation. Intuitively, we would like to explicitly communicate that deleting a record is forbidden, most likely by raising an error. Let's create a NotDeletable module in which we override the ActiveRecord methods.

```
module NotDeletable
  def delete
    raise 'Deleting a record is forbidden!'
  end
end
```

and do the same for `delete!`, `destroy`, `destroy!`. Even better, we could define these methods dynamically in order to save some space.

```
module NotDeletable
  [:delete, :delete!, :destroy, :destroy!].each do |method_name|
    define_method method_name, lambda { raise 'Deleting a record is forbidden!' }
  end
end
```

after including it to our `Model` ActiveRecord subclass we can see it works pretty neat.

object = Model.create
object.delete

will indeed result in an exception

```
RuntimeError: Destroying Model is forbidden!

```
The same goes when we call other methods defined by us.

The problem seems to be solved. However, these are not the only methods by which we will invoke the DELETE statement in the database. While `destroy_all` will raise an exception as expected, calling `delete_all` on an ActiveRecord::Relation instance will happily delete our records.

```
Model.where(id: 1).delete_all
  SQL (2.7ms)  DELETE FROM "models" WHERE "models"."id" = $1  [["id", 1]]
=> 1
Model.find(1)
ActiveRecord::RecordNotFound: Couldn't find Model with 'id'=1
```

Quite embarassing. However, a quick look into the docs will give us a good enough explanation.

"[`destroy_all`] Destroys the records matching conditions by instantiating each record and calling its destroy method ... If you want to delete many rows quickly, without concern for their associations or callbacks, use `delete_all` instead."
(http://apidock.com/rails/ActiveRecord/Relation/destroy_all)

As we can see, a more general solution is required. One way to solve this is to monkey-patch the ActiveRecord::Relation which responds to `delete_all`

```
module ActiveRecord
  class Relation
    if @klass == self.class
      def delete_all
        raise 'Deleting a record is forbidden!'
      end
    end
  end
end
```


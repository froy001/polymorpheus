# Polymorpheus
**Polymorphic relationships in Rails that keep your database happy with almost no setup**

### Background
* **What is polymorphism?** [Rails Guides has a great overview of what polymorphic relationships are and how Rails handles them](http://guides.rubyonrails.org/association_basics.html#polymorphic-associations)

* **If you don't think database constraints are important** then [here is a presentation that might change your mind](http://bostonrb.org/presentations/databases-constraints-polymorphism). If you're still not convinced, this gem won't be relevant to you.

* **What's wrong with Rails' built-in approach to polymorphism?** Using Rails, polymorphism is implemented in the database using a `type` column and an `id` column, where the `id` column references one of multiple other tables, depending on the `type`. This violates the basic principle that one column in a database should mean to one thing, and it prevents us from setting up any sort of database constraint on the `id` column.


## Basic Use

We'll outline the use case to mirror the example [outline in the Rails Guides](http://guides.rubyonrails.org/association_basics.html#polymorphic-associations): 

* You have a `Picture` object that can belong to an `Imageable`, where an `Imageable` is a polymorphic representation of either an `Employee` or a `Product`.

With Polymorpheus, you would define this relationship as follows:

**Database migration**

```
class SetUpPicturesTable < ActiveRecord::Migration
  def self.up
    create_table :pictures do |t|
      t.integer :employee_id
      t.integer :product_id
    end
  
    add_polymorphic_constraints 'pictures',
      { 'employee_id' => 'employees.id',
        'product_id' => 'products.id' }
  end
  
  def self.down
    remove_polymorphic_constraints 'pictures',
      { 'employee_id' => 'employees.id',
        'product_id' => 'products.id' }  
        
    drop_table :pictures
  end
end
```

**ActiveRecord model definitions**

```
class Picture < ActiveRecord::Base
  belongs_to_polymorphic :employee, :product, :as => :imageable
  validates_polymorphic :imageable
end

class Employee < ActiveRecord::Base
  has_many :pictures
end

class Product < ActiveRecord::Base
  has_many :pictures
end
```

That's it!

Now let's review what we've done.


## Database Migration

* Instead of `imageable_type` and `imageable_id` columns in the pictures table, we've created explicit columns for the `employee_id` and `product_id`
* The `add_polymorphic_constraints` call takes care of all of the database constraints you need, without you needing to worry about sql! Specifically it:
  * Creates foreign key relationships in the database as specified. So in this example, we have specified that the `employee_id` column in the `pictures` table should have a foreign key constraint with the `id` column of the `employees` table.
  * Creates appropriate triggers in our database that make sure that exactly on or the other of `employee_id` or `poduct_id` are specified for a given record. An exception will be raised if you try to save a database record that contains both or none of them.
* **Options for migrations**: There are options to add uniqueness constraints, customize the foreign keys generated by Polymorpheus, and specify the name of generated database indexes. For more info on this, [read the wiki entry](https://github.com/wegowise/polymorpheus/wiki/Migration-options).

## Model definitions

* The `belongs_to_polymorphic` declaration in the `Picture` class specifies the polymorphic relationship. It provides all of the same methods that Rails does for it's built-in polymorphic relationships, plus a couple additional features. See the Interface section below.
* `validates_polymorph` declaration: checks that exactly one of the possible polymorphic relationships is specified. In this example, either an `employee_id` or `product_id` must be specified -- if both are nil or if both are non-nil a validation error will be added to the object.
* The `has_many` declarations are just normal Rails declarations.


## Requirements / Support

* Currently the gem only supports MySQL. Postgres is a much better database for people who care about databases. We know this, and **support for Postgres is on its way**.
* This gem is tested and has been used in production under Rails 2.3.8 and 3.0.7. 
* Rails 3.1 should work, but you'll still need to use `up` and `down` methods in your migrations.

## Interface

The nice thing about Polymorpheus is that under the hood it builds on top of the Rails conventions you're already used to which means that you can interface with your polymorphic relationships in simple, familiar ways. There are also a number of helpful interface helpers that you can use. 

Let's use the example above to illustrate.

```
sam = Employee.create(name: 'Sam')
nintendo = Product.create(name: 'Nintendo')

pic = Picture.new
 => #<Picture id: nil, employee_id: nil, product_id: nil>

pic.imageable
 => nil

# the following two options are equivalent, just as they are normally with ActiveRecord:
#   pic.employee = sam
#   pic.employee_id = sam.id

# if we specify an employee, the imageable getter method will return that employee:
pic.employee = sam;
pic.imageable
 => #<Employee id: 1, name: "Sam">
pic.employee
 => #<Employee id: 1, name: "Sam">
pic.product
 => nil

# if we specify a product, the imageable getting will return that product:
Picture.new(product: nintendo).imageable
 => #<Product id: 1, name: "Nintendo">

# but if we specify an employee and a product, the getter will know this makes no sense and return nil for the imageable:
Picture.new(employee: sam, product: nintendo).imageable
 => nil

# There are some useful helper methods:

pic.imageable_active_key
 => "employee_id"

pic.imageable_query_condition
 => {"employee_id"=>"1"}
 
pic.imageable_types
 => ["employee", "picture"]
 
Picture::IMAGEABLE_KEYS
 => ["employee_id", "picture_id"]
```

# CRUDing Related Models — Self References, 1:n

<%= include_partial("_header.md", {s_one: true}) %>

## Self References: a User has many Users as followers

Often we will need to self-reference a model to define relationships 
between instances of the model. 

![ERD][erd-self-ref-1n]

While this resembles a __1:n__ relationship, it will act much more 
similarly to a __n:n__ relationship with a through table.

This 1:n relationship has a `through` relationship to another model to 
keep track of the relationships between instances of the __same__ model.

- **[Understanding and Context](#understanding-and-context)**
- **[Migrations](#migrations)**
- **[Models](#models)**
- **[Routes](#routes)**

### Understanding and Context

In this *1:n* relationship, one instance of the model will have many 
instances of the same model. Ideally, it would look like this:

![Ideal ERD][erd-self-ref-idea-1n]

__Yet in a self referenced 1:n relationship, there is an added 
complexity due to a join table between the same model.__ The join table 
will hold the related ids while defining each id's characteristics in 
the table.

![ERD][erd-self-ref-1n]

While the latter seems more complicated, we'll actually find that the 
added table will speed up our database calls and make our work easier
in the future.

Below we will see how to create the migrations for the `:through` table,
but we will not connect the two tables until later, in the models.

### Migrations

Migrations are a bit complicated, as we'll have to go beyond the 
terminal generate command. However, we'll start in the same place:

`$ rails generate model Follow follwer_id:integer followed_id:integer`

Here we've created a join-table that's dependent on the same User model,
however, __notice we are NOT using a reference within this table.__

We'll make that association in the model in the next section.

Now we have a migration that looks something like this:

```ruby
class CreateFollows < ActiveRecord::Migration
  def change
    create_table :follows do |t|
      t.integer :follower_id
      t.integer :followed_id

      t.timestamps null: false
    end
  end
end
```

However, we'll also need to add an index on each column to make locating
the relationship more efficiently.

```ruby
class CreateFollows < ActiveRecord::Migration
  def change
    create_table :follows do |t|
      t.integer :follower_id
      t.integer :followed_id

      t.timestamps null: false
    end
    add_index :follows, :follower_id
    add_index :follows, :followed_id
    add_index :follows, [:follower_id, :followed_id], unique: true
  end
end
```

We then create the follows table in the usual fashion:

`$ rake db:migrate`

We will now need to write our *1:n* relationship within both our `User` 
model and our `Follow` model.

### Models

...

### Routes

...

<!-- LINKS -->

<%= include_partial("_links.md") %>

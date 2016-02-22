# CRUDing Related Models — Self References, n:n


<%= include_partial("_header.md", {s_n: true}) %>

## Self References: a User habtm Users as friends

Many CRUD apps need users to friend each other, making mutual connections
between one another.  Unlike the previous Follow `1:n` relationship, the
Friendship model will be a **symmetrical**/**bi-directional** self-referencing
association.

A good way to think of this is that, when "friending", _we create and destroy
mutual relationships._ Unlike the `1:n`, we do not differentiate between
two kinds of relationship. Both influencing each other in the same way,
containing multiple friendships.

- **[Understanding and Context](#understanding-and-context)**
- **[Migrations](#migrations)**
- **[Models](#models)**
- **[Routes](#routes)**

### Understanding and Context

Unlike following, friendship is symmetrical, therefore creating a
has_many_and_belongs_to relationship within one model. You may think we can
simply use a habtm relationship to itself:

![wrong ERD][erd-self-ref-idea-nn]

But, of course, that makes a nightmare of methods and terribly inefficient
database queries.

As with the previous self-referencing relationship, we'll of course use a join
table.

![correct ERD][erd-self-ref-nn]

Notice that unlike the `1:n`, we'll need two attachments to the same model.
Due to the symmetrical nature of friendships we will need a more complex join
table than the previous 1:n.

We must specify the name of the join table, as well as the joining
columns just like the `1:n` self-referential relationship. In addition, we'll
need to create a flip-flop of the index, as we may need to locate friend's
friends in our site as well.

### Migrations

In order to make our migration, let's go ahead and create the model first -
we'll add to it later.

`$ rails g model Friendship user_id:integer friend_id:integer`

This migration will only store two user_ids, however, it will store them in both
directions. This gives us the ability to find friends and friends of friends.
Additionally, the symmetrical indices will greatly speed up our database
queries.

Let's change the migration file and add in those indices!

```ruby
class CreateFriendships < ActiveRecord::Migration
  def change
    create_table :friendships do |t|
      t.integer :user_id
      t.integer :friend_user_id

      t.timestamps null: false
    end

    add_index :friendships, [:user_id, :friend_user_id], unique: true
    add_index :friendships, [:friend_user_id, :user_id], unique: true
  end
end
```

Now that we have a way of finding our Friendships within our database we're
ready to migrate.

`rake db:migrate`

Now, lets handle the relationships between our User model and Friendship model.

### Models

...

### Routes

...

<!-- LINKS -->

<%= include_partial("_links.md") %>

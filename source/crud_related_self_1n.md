# CRUDing Related Models — Self References, 1:n

<%= include_partial("_header.md", {s_one: true}) %>

## Self References: a User has many Users as followers

Often we will need to self reference a model and define relationships
between instances of the same model.

![ERD][erd-self-ref-1n]

While this resembles a __1:n__ relationship, it will act much more
similarly to a __n:n__ relationship with a through table.

This 1:n relationship has a `through` relationship to another model to
keep track of the relationships between instances.

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
but we will not connect the two tables until later in the models.

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

Notice the *multiple-key index* at the end of the migration - this
ensures a User cannot follow a user more than once (let's keep the
obsessive stalkers at bay!). Because we will be finding the relationship
based on User ids, we should add an index on EACH column for efficiency.

> A database index is a data structure that improves the speed of
> operations in a table. Indexes can be created using one or more
> columns, providing the basis for both rapid random lookups and
> efficient ordering of access to records.

We then create the follows table in the usual fashion:

`$ rake db:migrate`

We will now need to write our *1:n* relationship within both our `User`
model and our `Follow` model.

### Models

In order to understand our model, recall that the relationship
between Followers and Followings is a 1:n relationship, and therefore
asymetrical. Additionally, think of what must be created and destroyed
at the a follow or unfollow.

Our join table is holding the information that defines a relationship
between "active" following and passive "follower".

![following relationship](https://softcover.s3.amazonaws.com/636/ruby_on_rails_tutorial_3rd_edition/images/figures/user_has_many_following_3rd_edition.png)

Now, since each user is uniquely identified by `:followed_id`, we can
use `:follower_id` to show `active_follows` and `:followed_id` as a
`passive_follows`.

```ruby
class User < ActiveRecord::Base

  #...
  has_many :active_follows,  class_name:  "Follow",
                             foreign_key: "follower_id",
                             dependent:   :destroy
  has_many :passive_follows, class_name:  "Follow",
                             foreign_key: "followed_id",
                             dependent:   :destroy
```

Now let's create an association within the `Follow` model with a few
validations:

```ruby
class Follow < ActiveRecord::Base

  #...
  belongs_to :follower,  class_name: "User"
  belongs_to :followed, class_name: "User"
  validates :follower_id, presence: true
  validates :followed_id, presence: true
```

This is looking great, but we're not done! We need to implement the through
relationship. We can now create a "following" group for "followed", and
"followers" for our "follower".

```ruby
class User < ActiveRecord::Base

  #...
  has_many :active_follows,  class_name:  "Follow",
                             foreign_key: "follower_id",
                             dependent:   :destroy
  has_many :passive_follows, class_name:  "Follow",
                             foreign_key: "followed_id",
                             dependent:   :destroy
  has_many :following, through: :active_follows,  source: :followed
  has_many :followers, through: :passive_follows
```

Notice the `:followers` does not have a `source:` - this is because Rails will
complete this plural for us. "Followeds" simply sounds strange, so we change the
term.

Last, we'll need to create a few utility methods: `user.follow(other_user)`,
`user.unfollow(other_user)`, and `user.following?(other_user)` as a boolean
check. These will be useful later on, in following and unfollowing other users.

```ruby
class User  < ActiveRecord::Base

  #...
  # Follows a user.
  def follow(other_user)
    active_relationships.create(followed_id: other_user.id)
  end

  # Unfollows a user.
  def unfollow(other_user)
    active_relationships.find_by(followed_id: other_user.id).destroy
  end

  # Returns true if the current user is following the other user.
  def following?(other_user)
    following.include?(other_user)
  end
  #...
end
```

### Routes

We now need to include the routes for our following and followers action.

Here's what we're looking for:

| HTTP Request |         URL         |   Action  |        Named Route       |
|:------------:|:-------------------:|:---------:|:------------------------:|
| GET          | users/:id/following | following | following_user_path(:id) |
| GET          | users/:id/followers | followers | following_user_path(:id) |

In order to acheive this kind of nested routes, we'll need to manipulate the
resources routes for `:users`.

```ruby
Rails.application.routes.draw do

  #...
  resources :users do
    member do
      get :following, :followers
    end
  end
end
```

The `member` method comes from Rails - it arranges for the routes to respond
as planned.  You can think of it as a routes specific `.each` method.

We'll also need to make routes for creating and destroying follows.

```ruby
Rails.application.routes.draw do

  #...
  resources :users do
    member do
      get :following, :followers
    end
  end
  resources :follows,  only: [:create, :destroy]
end
```

Now that our routes are in place, we'll need to make the corresponding methods
in our `UsersController`.

Let's begin by adding the `:following` and `:followers` methods to our
`UsersController`. We'll also ensure these methods will only be available to a
logged in user.

```ruby
class UsersController < ApplicationController
  before_action :logged_in_user, only: [:index, :edit, :update, :destroy,
                                        :following, :followers]
  #...
  def following
    #... your logic may look differently here
    @title = "Following"
    @user  = User.find(params[:id])
    @users = @user.following.paginate(page: params[:page])
    render 'show_follow'
  end

  def followers
    #... your logic may look differently here
    @title = "Followers"
    @user  = User.find(params[:id])
    @users = @user.followers.paginate(page: params[:page])
    render 'show_follow'
  end

  #...
end
```

You may need the `will_paginate` gem to use the paginate method, however,
depending on your site's requirements, your logic may differ. Essentially,
we're getting our user, and retrieving the followers in paginated (numbered)
form. We're then rendering some kind of show page for the follow.

We also need to define the `:create` and `:destroy` methods on the
`FollowsController`. In this controller, we'll limit all actions by checking for
a logged in user.

```ruby
class RelationshipsController < ApplicationController
  before_action :logged_in_user

  def create
    user = User.find(params[:followed_id])
    current_user.follow(user)
    redirect_to user
  end

  def destroy
    user = Follow.find(params[:id]).followed
    current_user.unfollow(user)
    redirect_to user
  end
end
```

You can now follow and unfollow another user on your site! We  access the
specific user by finding the id in the params, and sending it to one of the
utility instance methods we created earlier.

The rest is up to you! You now have everything you need to define a
self-referencing `1:n` or asymmetrical self-referential relationship. Use these
methods to assist you in displaying information in your view!

<!-- LINKS -->

<%= include_partial("_links.md") %>

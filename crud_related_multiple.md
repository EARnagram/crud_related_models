# CRUDing Related Models — Multiple, Named Relationships

While we went in to CRUDing in depth with a single model, and we explored
what it means to link, or **relate**, our models, we haven't shown you an
example of some of the tools Rails gives us to CRUD related models.

To that end, we have four examples/walkthroughs to show you about
how to structure the models, views, controllers and routes when trying
to CRUD resources whose structure depends upon their relations:

1.  [**1:n (one-to-many) relationships**: *a `Shelf` has many `Book`s*.][crud-1n]
2.  [**n:n (many-to-many) relationships**: *a `Topic` has and belongs to many `Book`s*.][crud-nn]
3.  **multiple, named relationships**: *a `User` has many `Book`s they've created,*
    *and habtm `Book`s they've "favorited."*
4.  **self referential relationships**:
    - ***(1:n)*** *a `User` has many `User`s as* ***followers***, and
    - ***(n:n)*** *a `User` has and belongs to many `User`s as* ***friends***.

## Multiple, Named Relationships

We often will have to relate the data in two separate models to
one another via more than one relationship. A clear example of 
this is in applications where users create and share information,
interacting with one another's creations.

For our example here, we will use:

![Basic ERD][erd-basic]

A way to describe this is: users create books, and users also
favorite books that other users (or themselves) have created.

- **[Understanding and Context](#understanding-and-context)**
- **[Migrations](#migrations)**
- **[Models](#models)**
- **[Routes](#routes)**

### Understanding and Context

As is often the case, the relationships are not just different
in purpose, they are different in type! The two models here
are related in 1:n and n:n ways. If we normalize the n:n
relationship here, we will see:

![Normalized ERD][erd-basic-normalized]

We already know that this relationship, when defined in a relational
database, needs to be **normalized**, or turned into a series of 1:n
relationships. For the example, this would look like:

And we know from the n:n walkthrough, that there are two ways to
think about this relationship, as a `habtm` or as a `has_many :through`.

The two options, next to each other look like:

![Normalized ERD][erd-complete]

Below we will take a look at how to set up the two 
different approaches (`habtm` and `has_many :through`) in
the migrations and models. Beyond that, all interaction
works the same (depending on the relationship) as in the
previous walkthroughs. We will, however, see an example
set up of the necessary routes to allow rich data interactions.

[Rails Guides to models links...][]

### Migrations

As stated above, both of the two "first-class" models in an n:n
relationship are independent. The dependent entity bears the weight
for holding the data about the related tables, and is where you need
to focus attention.

For both of the formats, you could generate your models like this:

```
$ rails g model Topic name
$ rails g model Post title body:text
```

#### `habtm`

To use the simpler `habtm` form of n:n relationships, you need to use
that Rails naming magic! In this case, you create a table called a
**join table** that holds the relationships. **A join table is named 
like this: the two entities' *table names* in alphabetical order.**

Thus, for the above, we would create the migration:

```
$ rails g migration CreatePostsTopics post:references topic:references
```

**Note: we are generating just the migration, not the model, and use the
migration naming style for it!**

The output of the migration generator looks like:

```ruby
class CreatePostsTopics < ActiveRecord::Migration
  def change
    create_table :posts_topics do |t|
      t.references :post, index: true, foreign_key: true
      t.references :topic, index: true, foreign_key: true
    end
  end
end
```

See the [1:n examples][crud-1n] for more detail on how to create 
`belongs_to`-style relationships in tables with the Rails generators.

#### `has_many :through`

To use the somewhat more complicated, but more explicit, 
`has_many :through` form of n:n relationships, you can do *the exact 
same thing* with the database, but also generate a model. Ie:

```
$ rails g model PostsTopics post:references topic:references
```

However, since we don't need to use the Rails naming magic that makes
the relationship work (we instead are declaring the name of the 
ActiveRecord model that relates the two together), we will often give
the join table a better name if we can. Here, we are using this as
an example:

```
$ rails g model Category post:references topic:references
```

**Note: we are once again generating a model AND migration, not just a
migration!** We therefore use the model name style in the generator
instead of the migration name style.

### Models

#### `habtm`

Here we see the *real* power of this format! Once creating the above
migration, all we need to do is [add the following to the models][rg-habtm]:

**`/app/models/topic.rb`:**

```ruby
class Topic < ActiveRecord::Base
  has_and_belongs_to_many :posts
end
```

**`/app/models/post.rb`:**

```ruby
class Post < ActiveRecord::Base
  has_and_belongs_to_many :topics
end
```

After which you can use the following ActiveRecord commands:

```ruby
post = Post.find(1)
post.topics
#=> [<Topic>, …]
```

… and vice-versa.

#### `has_many :through`

If we generated the join-table model correctly, it will be ready to go:

**`/app/models/post_topic.rb`** or **`/app/models/category.rb`**:

```ruby
class PostTopic < ActiveRecord::Base
  belongs_to :post
  belongs_to :topic
end

# or …

class Category < ActiveRecord::Base
  belongs_to :post
  belongs_to :topic
end
```

What we need to add to, then, are the n:n-related models! First we
implement their relationship with the join-table model (which below
we will assume is `Category`), then we relate through the join-table 
model to the other target:

**`/app/models/topic.rb`:**

```ruby
class Topic < ActiveRecord::Base
  has_many :categories
  has_many :posts, through: :categories
end
```

**`/app/models/post.rb`:**

```ruby
class Post < ActiveRecord::Base
  has_many :categories
  has_many :topics, through: :categories
end
```

**Note: the order is important!** Finish the simple 1:n relationship
(`has_many`/`belongs_to`) with the join-table model, then build a
`has_many :through` through that relationship.

### Routes

For the most part, since the related, "first-class" models are 
independent, they can exist side-by-side in the routes.

**`/config/routes.rb`:**

```ruby
  resources :posts
  resources :topics
```

If you wanted to add or remove a topic from a post, for example, that
would simply be an update to your post, ie `PUT /posts/1` with the
post's form's data…

The one place in which it may make sense to nest (relate) these routes
is with index: ie, list (index) all the topics in a post, or list (index)
all the posts in a topic. This can be done with:

```ruby
  resources :posts do
    resources :topics, only: [:index]
  end
  resources :topics do
    resources :posts, only: [:index]
  end
```

The routes would be:

```
GET /posts/1/topics # all the topics in a post
GET /topics/1/posts # all of the posts in a topic
```

To explore nested routes further, [check out the Rails Guides][rg-routes].

<!-- LINKS -->

[crud-1n]: /crud_related_1n.md
[crud-nn]: /crud_related_nn.md
[crud-mu]: /crud_related_multiple.md
[crud-se]: /crud_related_self.md

[erd-basic]:            /assets/img-crud-related-multiple-basic.jpg
[erd-basic-normalized]: /assets/img-crud-related-multiple-basic-normalized.jpg
[erd-complete]:         /assets/img-crud-related-multiple-complete.jpg

[rg-routes]: http://guides.rubyonrails.org/routing.html#nested-resources

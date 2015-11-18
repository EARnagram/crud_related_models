# CRUDing Related Models — n:n

While we went in to CRUDing in depth with a single model, and we explored
what it means to link, or **relate**, our models, we haven't shown you an
example of some of the tools Rails gives us to CRUD related models.

To that end, we have four examples/walkthroughs to show you about
how to structure the models, views, controllers and routes when trying
to CRUD resources whose structure depends upon their relations:

1.  [**1:n (one-to-many) relationships**: *a `Shelf` has many `Book`s*.][crud-1n]
2.  **n:n (many-to-many) relationships**: *a `Topic` has and belongs to*
    *many `Book`s*.
3.  [**multiple named relationships**: *a `User` has many `Book`s they've created,*
    *and habtm `Book`s they've "favorited."*][crud-mu]
4.  **self referential relationships**:
    - [***(1:n)*** *a `User` has many `User`s as* ***followers***][crud-s1], and
    - [***(n:n)*** *a `User` has and belongs to many `User`s as* ***friends***][crud-sn].

## Many-to-many (*n:n*)

Quite often we need to relate models to one another not in the simple,
**1:n** (one-to-many) format, but in the more complex **n:n** 
(many-to-many) format. For our example here, we will use:

![ERD][erd-nn]

We already know that this relationship, when defined in a relational
database, needs to be **normalized**, or turned into a series of 1:n
relationships. For the example, this would look like:

![Normalized ERD][erd-normal]

This **needs to be done at the database level**, no matter what.

However, Rails allows us to define these relationships on the models
(the internal, ActiveRecord representation of the database) in two
different ways:

1.  A **`has_and_belongs_to_many`** (or **`habtm`**) relationship that 
    is "bi-directional" between the two first-class models (`Post` and 
    `Topic`, the models with ActiveRecord classes), or …
2.  by "upgrading" the join table (`posts_topics`) to a first class
    model. This model could be called `PostTopic`, or the table and the
    model can be given their own name, for example `categories` and
    `Category`, like:

    ![Link-through ERD][erd-thru]
    
    Now you define the standard 1:n relationships (*a post `has_many` 
    categories, and a category `belongs_to` a post*, and *a topic 
    `has_many` categories, and a category `belongs_to` a topic*). Finally 
    you add a new type of relationship, the **`has_many, :through`** to 
    define the relationships between `Post` and `Topic`.

Whenever possible we will use the first (it's simpler). However, if
necessary, we may use the second. To know which is right, read 
**Understanding and Context** below, or 
[check out the Rails Guides][rg-assoc].

- **[Understanding and Context](#understanding-and-context)**
- **[Migrations](#migrations)**
- **[Models](#models)**
- **[Routes](#routes)**
- **[Controllers](#controllers)**
- **[View Templates & Forms](#view-templates--forms)**

### Understanding and Context

n:n relationships are bi-directional, in that the two related models
are both stand-alone models and can be CRUDed independently, and that
the relationship is the same for each constituent model. However, there
is always an extra entity (**join table**) in an n:n relationship!

If the two "first-class" models in the relationship are related to
eachother "directly," ie *all you want to track is the fact that they
are related*, then you can simplify the relationship by using
`has_and_belongs_to_many` (**`habtm`**). Keep in mind, tho, that while
the models' relationship is straightforward, you still need to create
the underlying entities' table.

However, if you find that:

- the relationship has any data associated with it (a comment, for
  example, as opposed to a "like"),
- the join table is already a "first-class" model (an ActiveRecord 
  class) itself,
- the two, related "first-class" models are related in more than one
  way (you can't have multiple, named, n:n routes),

then you will have to use **`has_many :through`**.

Note: this is not the only reason you would use `… :through`! It
is one of a few use-cases for this Rails structure. For others, 
[see the Rails Guides][rg-thru].

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

### Controllers

Once again, both of the two "first-class" models in an n:n relationship
are independent, and don't need to change their routes or their
actions.

The special cases are often for indexing, creating or updating.

#### posts#index

Indexing is mostly an issue with checking whether or not it is a nested 
route:

```ruby
  def index
    if params[:topic_id]
      @topic = Topic.find params[:topic_id] 
      @posts = @topic.posts
    else
      @post = Post.all
    end
  end
```

#### posts#update

Given a request to remove a topic from a post, and add two more
(remove `science`, and add `history` and `politics`, eg):

```ruby
  # We can assume that we will receive the topic information
  # as an array of Topic IDs, as will be demonstrated in the
  # below view examples.
  def update
    @post = Post.find params[:id]

    if @post.update(post_params)
      flash[:notice] = "You've updated you're post!"
      redirect_to post_path(@path)
    else
      render :edit
    end
  end

  private

    # Whitelisting, or allowing specific params!
    # NOTE: :topic_ids!!
    def post_params
      params.require(:post).permit(:title, :body, :topic_ids)
    end
```

#### posts#create

Given a request to create a post, add an existing topic to it (`music`)
and create and add a new topic to it (`music theory`):

```ruby
  def create
    # This is optional, of course, but it allows you to dynamically
    # generate new topics, and does not follow the simple example
    # above of just passing Topic IDs!
    unless params[:new_topic_names].empty?
      new_topics.each do |topic_name|
        topic = Topic.where(name: topic_name).first_or_create!
        params[:topic_ids] << topic.id
      end
    end

    # From here is the standard format, of recieving Topic IDs
    # and adding them. Add the above if you want to generate new
    # topics AND add them just by using the Post form!
    @post = Post.new post_params

    if @post.save
      flash[:notice] = "You've created a new post!"
      redirect_to post_path(@path)
    else
      render :new
    end
  end

  private

    # Whitelisting, or allowing specific params!
    # NOTE: :topic_ids!
    def post_params
      params.require(:post).permit(:title, :body, :topic_ids)
    end

    # Split the new topics into an array, and then strip
    # any leading or trailing whitespace from each
    def new_topics
      params[:new_topic_names].split(",").map(&:strip)
    end
```

### View Templates & Forms

Most of the issues that arise when attempting to relate models in an
n:n format come from how to structure the request data in forms. You
need to be able to reference data in the database, and send it in a
standard format to the server!

To this end, different methods for structuring the forms, and the 
built in Rails form and tag helpers to cnstruct them, are discussed
below.

For a reminder of the basic HTML underlying these forms, 
[see the HTML Forms Cheatsheet][html-forms].

#### Checkboxes, Multi-selects & Options, and Multi-model Forms

One of the more confusing parts of working with Rails is the
profusion of form and input tag helpers, and the unclear documentation
around using them. We'll try to break down some best practices here,
since using `<select>` tags (dropdowns and multi-selects) or 
`<input style="checkbox">` checkboxes is essential to forms where you
n:n relate independent resources.

The general rules are this:

- if you want people to be able to choose many choices (eg: tags or 
  topics), you should use a multi-select or checkboxes;
- if you want people to be able to choose only one choice from among
  many (eg: one sub-category from each of a few wider categories), you 
  use a dropdown or radio button.

Dropdowns and multi-selects are represented as `<select>` and `<option>`
tags, while radio buttons and checkboxes are both `<input>` tags.

Below we are only going to show checkboxes and multi-selects, since they
are the more complex examples and apply better to n:n relationships.
There are many other reasons to use these form tags, and these
examples can help you understand how to use them!

For all of the below, we assume the controller looks like this:

**`/app/controllers/posts_controller.rb`:**

```ruby
  def new
    @post = Post.new
    @topics = Topic.order(:name) # list all topics, ordered by name
  end
```

#### Checkboxes

This uses a built-in "form options helper" (note that it **doesn't** use
`f.collection_check_boxes`), and more information on how to use these 
[can be found in the Rails API][ra-formopts].

**`/app/views/books/new.html.erb`**, **`/app/views/books/edit.html.erb`**
or **`/app/views/books/_form.html.erb`:**

```html
  <%= form_for @post do |f| %>
    <!-- … -->
    <div class="field">
      <div class="input-group">
        <h5>Topics</h5>
        <%= collection_check_boxes :post, :topic_ids, @topics, :id, :name %>
    </div>
    <!-- … -->
  <% end %>
```

For more general ways to create checkboxes using `form_for#check_box` a
and similar helpers, [check out the Rails API][ra-ff-check].

**Note: if you are using `form_tag` instead of `form_for`**, you need to 
[check out the `form_tag` Rails API instead][ra-tag-check].

#### Multi-selects (aka Selects/Dropdowns with Multiple Selections)

This uses a built-in "form options helper" (note that it **doesn't** use
`f.collection_select`), and more information on how to use these 
[can be found in the Rails API][ra-formopts].

```html
  <%= form_for @post do |f| %>
    <!-- … -->
    <div class="field">
      <div class="input-group">
        <h5>Topics</h5>
        <%= collection_select :post, :topic_ids, @topics, :id, :name, {}, multiple: true %>
      </div>
    </div>
    <!-- … -->
  <% end %>
```

**Note: very often, you will want to use `<select>` and `<option>` tags 
in a different way!**

Be sure to [peruse the Rails API for form options helpers][ra-formopts] 
for `options_for_select` and `options_from_collection_for_select`, both 
of which need to be used with `select` or `select_tag` helper.


#### Multi-model Forms

Sometimes we want users to be able to add new instances of another,
related resource right from the form of the current resource. For 
example, we write a post about music, and realize there is no topic
for `music theory` to add to it.

We can also send this data, and have a nice, interactive form, using
many different methods, notably JavaScript.

A very simple, non-JavaScript example is given below, where we allow
the user to add new tags, comma-separated, to an input field in a form.
The text, when submitted, will be split in the controller and turned
in to topic objects that are added to the topic list.

```html
  <%= form_for @post do |f| %>
    <!-- … -->
    <div class="field">
      <%= label :new_topic_names, "New topics, comma-separated" %><br>
      <%= text_field_tag :new_topic_names %>
    </div>
    <!-- … -->
  <% end %>
```

**Note: we are using the tag helpers, not the `form_for` ones!** There
is no `f.label` or `f.text_field`, for example.

And remember the `posts#create` from above? This is why we need it:

**`/app/controllers/posts_controller.rb`:**

```ruby
  def create
    unless params[:new_topic_names].empty?
      new_topics.each do |topic_name|
        topic = Topic.where(name: topic_name).first_or_create!
        params[:post][:topic_ids] << topic.id
      end
    end

    # …
  end

  private

    # …

    def new_topics
      params[:new_topic_names].split(",").map(&:strip)
    end
```

<!-- Remember, entities are named like this (given an entity of 
*secret places*, for example):

- HTTP resource:    `/secret_places`  (plural snake-case)
- router method:    `:secret_places`  (plural snake-case symbol)
- controller name:  `secret` ()
- model class name: `SecretPlace`     (singular camel-case)
- model file name:  `secret_place.rb` (singular snake-case)
- database table:   `secret_places`   (plural snake-case) -->

<!-- LINKS -->

[alias]: #alias

[crud-1n]: crud_related_1n.md
[crud-nn]: crud_related_nn.md
[crud-mu]: crud_related_multiple.md
[crud-s1]: crud_related_self_1n.md
[crud-sn]: crud_related_self_nn.md

[erd-1n]:               assets/img-crud-related-1n.jpg
[erd-1n-user]:          assets/img-crud-related-1n-user.jpg
[erd-nn]:               assets/img-crud-related-nn.jpg
[erd-normal]:           assets/img-crud-related-nn-normalized.jpg
[erd-thru]:             assets/img-crud-related-nn-through.jpg
[erd-basic]:            assets/img-crud-related-multiple-basic.jpg
[erd-basic-normalized]: assets/img-crud-related-multiple-basic-normalized.jpg
[erd-complete]:         assets/img-crud-related-multiple-complete.jpg

[rg-routes]:        http://guides.rubyonrails.org/routing.html#nested-resources
[rg-routes-custom]: http://guides.rubyonrails.org/routing.html#customizing-resourceful-routes
[rg-assoc]:         http://guides.rubyonrails.org/association_basics.html#choosing-between-has-many-through-and-has-and-belongs-to-many
[rg-has-many]:      http://guides.rubyonrails.org/association_basics.html#the-has-many-association
[rg-belongs-to]:    http://guides.rubyonrails.org/association_basics.html#the-belongs-to-association
[rg-habtm]:         http://guides.rubyonrails.org/association_basics.html#the-has-and-belongs-to-many-association
[rg-thru]:          http://guides.rubyonrails.org/association_basics.html#the-has-many-through-association
[rg-ar-assoc]:      http://guides.rubyonrails.org/association_basics.html#detailed-association-reference

[ra-formopts]:  http://api.rubyonrails.org/classes/ActionView/Helpers/FormOptionsHelper.html
[ra-tags]:      http://api.rubyonrails.org/classes/ActionView/Helpers/FormTagHelper.html
[ra-ff]:        http://api.rubyonrails.org/classes/ActionView/Helpers/FormHelper.html
[ra-tag-check]: http://api.rubyonrails.org/classes/ActionView/Helpers/FormTagHelper.html#method-i-check_box_tag
[ra-ff-check]:  http://api.rubyonrails.org/classes/ActionView/Helpers/FormHelper.html#method-i-check_box
[ra-ar-assoc]:  http://api.rubyonrails.org/classes/ActiveRecord/Associations/ClassMethods.html
[ra-has-many]:  http://apidock.com/rails/ActiveRecord/Associations/ClassMethods/has_many

[html-forms]:   https://gist.github.com/h4w5/8848398
[so-post]:      http://stackoverflow.com/questions/21688200/rails-4-checkboxes-for-has-and-belongs-to-many-association
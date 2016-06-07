# CRUDing Related Models — Multiple, Named Relationships

While we went in to CRUDing in depth with a single model, and we explored
what it means to link, or **relate**, our models, we haven't shown you an
example of some of the tools Rails gives us to CRUD related models.

The [Rails Guides for ActiveRecord Associations](http://guides.rubyonrails.org/association_basics.html)
has great information on how to do this! You need to read them! But they 
don't walk you through each step of implementing them in an app.

To that end, we have four examples/walkthroughs to show you about
how to structure the models, views, controllers and routes when trying
to CRUD resources whose structure depends upon their relations:

1.  [**1:n (one-to-many) relationships**: *a `Shelf` has many `Book`s*.][crud-1n]
2.  [**n:n (many-to-many) relationships**: *a `Topic` has and belongs to*
    *many `Book`s*.][crud-nn]
3.  **multiple named relationships**: *a `User` has many `Book`s they've created,*
    *and habtm `Book`s they've "favorited."*
4.  **self referential relationships**:
    - [***(1:n)*** *a `User` has many `User`s as* ***followers***][crud-s1], and
    - [***(n:n)*** *a `User` has and belongs to many `User`s as* ***friends***][crud-sn].

# CRUDing Related Models

While we went in to CRUDing in depth with a single model, and we explored
what it means to link, or **relate**, our models, we haven't shown you an
example of some of the tools Rails gives us to CRUD related models.

## Multiple, Named Relationships

We often will have to relate the data in two separate models to
one another via more than one relationship. A clear example of
this is in applications where users create and share information,
interacting with one another's creations.

For our example here, we will use:

![Basic ERD][erd-basic]

A way to describe this is: **users create books, and users also
favorite books that other users (or themselves) have created.**

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

And we know from the n:n walkthrough, that there are two ways to
think about this relationship, as a `habtm` or as a `has_many :through`.
The two options, next to each other, look like:

![Normalized ERD][erd-complete]

As we can see, there's significantly less complexity for the `habtm`
form than with the `has_many :through` form… Generally, we would rather
use the former unless we have to use the latter; if there is any data
or logic associated with a favorite itself (if the *favorite* is instead
a *comment*, or a *tag*) for instance.

Below we will take a look at how to set up the two different approaches
(`habtm` and `has_many :through`) in the migrations and models. Beyond
that, all interaction works the same (depending on the relationship) as
in the previous walkthroughs. We will, however, see an example set-up of
the necessary routes to allow rich data interactions.

### Migrations

For both of the formats, you could generate your "first-class" models
like this:

```
$ rails g model User email name born_on:date
$ rails g model Book title description:text user:references
```

Remember to add `user:references`! This sets up your first, `has_many`/
`belongs_to` route.

#### `habtm`

Create the migration:

```
$ rails g migration CreateBooksUsers user:references book:references
```

**Note: the order of the names is important!** Alphabetical: in this
case, `books` before `users`. The output of the migration generator
looks like:

```ruby
class CreateBooksUsers < ActiveRecord::Migration
  def change
    create_table :books_users do |t|
      t.references :user, index: true, foreign_key: true
      t.references :book, index: true, foreign_key: true
    end
  end
end
```

See the [n:n examples][crud-nn] for more detail on creating such a
*join table*.

#### `has_many :through`

We would create the model:

```
$ rails g model Favorite user:references book:references
```

**Note: we are once again generating a model AND migration, not just a
migration!** We therefore use the model name style in the generator
instead of the migration name style.

See the [n:n examples][crud-nn] for more detail on generating
*join models*.

### Models

The major point here is to explore the ActiveRecord API a little further,
specifically the format of the
[`belongs_to`, `has_many`, and `habtm`][ra-ar-assoc] methods. Our example
will use the [`has_many` method][ra-has-many].

When we see:

```ruby
class Owner < ActiveRecord::Base
  has_many :puppies
end

bob = Owner.new
bob.puppies
#=> []
```

We **are not** saying *Relate the current model to the model named
`Puppy`*! This is a common misperception!

Instead, we are saying something more complex:

> Create a method on instances of the current model named `#puppies`.
> This method will return AR models that are related to the current
> model.
>
> If there is no option that names the AR model (class) this returns,
> assume the model's class name is `Puppy`.
>
> If there is no option that names the table to look to for the foreign
> key (relation information), assume the table's name is `puppies`.
>
> If there is no option that names column in the table to look to for
> the foreign key, assume the column's name is `owner_id`.

Thus: the symbol we pass to `has_many` is the ***name of the method it
creates***! Everything else is set to Rails conventional defaults:
model class name, table name, and foreign key column name. And we can
change any of them!

There is information about the options you can pass to these methods in
the API guides linked above, but even better examples are in the
**[Rails Guides][rg-ar-assoc]**.

#### `habtm`

For our specific example, we want to rename to relations, and otherwise
only need to pass options that tell ActiveRecord what the related model
is. With this information, Rails will reset the defaults for table name
(foreign key being the same).

**`/app/models/user.rb`:**

```ruby
class User < ActiveRecord::Base
  has_many :books
  has_and_belongs_to_many :favorites, class_name: "Book"
end
```

<div id="alias"></div>
**However, we have a problem!** the method that references the
`has_many` / `belongs_to` relationship may be ambiguous! Does it mean
*created* books, or *favorite* books?

```ruby
user.books
#=> ?????????
```

We have two options to fix this (if we find it a problem):

1.  **We can add an "alias,"** or another method that does the same
    thing as this method but has a more clear, semantic meaning, or …
2.  **we can rename this method,** like we did with the `hatbm` one.

We will start by **renaming** the method:

```
has_many :created_books, class_name: "Books"
```

If you like to create new books by using the syntax `#relation#create`
then just renaming the method could lead to some weird semantics:

```
user.books.create title: "The Great Gatsby", description: "boring!"
# vs, if you rename the basic method:
user.created_books.create title: "Moby Dick", description: "very boring!"
```

Thus, we will **alias** that method:

```ruby
class User < ActiveRecord::Base
  has_many :books
  has_and_belongs_to_many :favorites, class_name: "Book"

  alias_attribute :created_books, :books
end
```

And we can write:

```ruby
user = User.find(1)

user.books.create title: "The Great Gatsby", description: "boring!"
user.created_books
#=> [<Book @title="The Great Gatsby">, …]

user.favorites.create title: "Moby Dick", description: "very boring!"
user.favorites
#=> [<Book @title="Moby Dick">, …]
```

**`/app/models/book.rb`:**

```ruby
class Book < ActiveRecord::Base
  belongs_to :created_by, class_name: "User"
  has_and_belongs_to_many :favorited_by, class_name: "User"
end
```

After this you can use the following ActiveRecord commands:

```ruby
book = Book.find(1)

book.created_by
#=> <User A>

book.favorited_by
#=> [<User B>, …]
```

#### `has_many :through`

Assuming the migrations above ran correctly, we will have the models
`User`, `Book`, and `Favorite`.

See the [n:n `has_many :through` examples][crud-nn] for more detail on
the below syntax, if necessary.

**`/app/models/favorite.rb`**:

```ruby
class Favorite < ActiveRecord::Base
  belongs_to :user
  belongs_to :book
end
```

**`/app/models/user.rb`:**

```ruby
class User < ActiveRecord::Base
  has_many :books

  has_many :favorites
  has_many :favorite_books, through: :favorites,
                            source:  :book

  alias_attribute :created_books, :books
end
```

The `source` option here, instead of the `class_name` options, is used
because we are telling Rails that `Favorite` has a method `book` that
the `has_many :through` will call to access a given favorite's book.

[For more information on the `alias_attribute` method, see above.][alias]


**`/app/models/book.rb`:**

```ruby
class Book < ActiveRecord::Base
  belongs_to :created_by, class_name: "User"

  has_many :favorites
  has_many :favorited_by, through: :favorites,
                          source:  :user
end
```

After this you can use the following ActiveRecord commands:

```ruby
user = User.create email:   "pj@ga.co",
                   name:    "PJ",
                   born_on: Date.parse("9/10/1990")

user.books.create title: "The Great Gatsby", description: "boring!"
user.created_books
#=> [<Book @title="The Great Gatsby">]

book = Book.create title: "Moby Dick", description: "very boring!"

Favorite.create user: user, book: book
#=> <Favorite 1>

book.favorites
#=> [<Favorite 1>]

user.favorites
#=> [<Favorite 1>]

user.favorite_books
#=> [<Book @title="Moby Dick">]

book.favorited_by
#=> [<User @name="PJ">]
```

### Routes

Just like with the n:n routes, we'll want to be able to CRUD our
indepenedent models seperately, while also indexing based on their
`habtm` or `has_many :through` relationships:

**`/config/routes.rb`:**

```ruby
  resources :users do
    resources :books, only: [:index]
  end
  resources :books do
    resources :users, only: [:index]
  end
```

But now we have more semantic names for these relationships, and need
to differentiate between the types of relationships!

We can do this a similar way to above: using options to change the Rails
defaults. You can check out
[how to customize routes in the Rails Guides][rg-routes-custom].

```ruby
  resources :users do
    resources :created_books, only:       [:index],
                              controller: :books

    resources :favorites, only:       [:index],
                          controller: :books
  end

  resources :books do
    resources :favorited_by, only:       [:index],
                             controller: :users
  end
```

This sets up the following routes:

```
                 Prefix Verb   URI Pattern                             Controller#Action
     user_created_books GET    /users/:user_id/created_books(.:format) books#index
         user_favorites GET    /users/:user_id/favorites(.:format)     books#index
                  users GET    /users(.:format)                        users#index
                        POST   /users(.:format)                        users#create
               new_user GET    /users/new(.:format)                    users#new
              edit_user GET    /users/:id/edit(.:format)               users#edit
                   user GET    /users/:id(.:format)                    users#show
                        PATCH  /users/:id(.:format)                    users#update
                        PUT    /users/:id(.:format)                    users#update
                        DELETE /users/:id(.:format)                    users#destroy
book_favorited_by_index GET    /books/:book_id/favorited_by(.:format)  users#index
                  books GET    /books(.:format)                        books#index
                        POST   /books(.:format)                        books#create
               new_book GET    /books/new(.:format)                    books#new
              edit_book GET    /books/:id/edit(.:format)               books#edit
                   book GET    /books/:id(.:format)                    books#show
                        PATCH  /books/:id(.:format)                    books#update
                        PUT    /books/:id(.:format)                    books#update
                        DELETE /books/:id(.:format)                    books#destroy
```

As you can see, that sends them all to just the two controllers,
`UsersController` and `BooksController`. You may actually find it useful
to create extra controllers to encapsulate the logic, but that is up
to you!

<!-- LINKS -->

[alias]: #alias

[crud-1n]: crud_related_1n.md
[crud-nn]: crud_related_nn.md
[crud-mu]: crud_related_multiple.md
[crud-s1]: crud_related_self_1n.md
[crud-sn]: crud_related_self_nn.md

[erd-1n]:                assets/img-crud-related-1n.jpg
[erd-1n-user]:           assets/img-crud-related-1n-user.jpg
[erd-nn]:                assets/img-crud-related-nn.jpg
[erd-normal]:            assets/img-crud-related-nn-normalized.jpg
[erd-thru]:              assets/img-crud-related-nn-through.jpg
[erd-basic]:             assets/img-crud-related-multiple-basic.jpg
[erd-basic-normalized]:  assets/img-crud-related-multiple-basic-normalized.jpg
[erd-complete]:          assets/img-crud-related-multiple-complete.jpg
[erd-self-ref-1n]:       assets/img-crud-related-self-referential-1n.png
[erd-self-ref-idea-1n]:  assets/img-crud-related-self-referential-idea-1n.png
[erd-self-ref-nn]:       assets/img-crud-related-self-referential-nn.png
[erd-self-ref-idea-nn]:  assets/img-crud-related-self-referential-idea-nn.png

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
# CRUDing Related Models — 1:n

While we went in to CRUDing in depth with a single model, and we explored
what it means to link, or **relate**, our models, we haven't shown you an
example of some of the tools Rails gives us to CRUD related models.

To that end, we have four examples/walkthroughs to show you about
how to structure the models, views, controllers and routes when trying
to CRUD resources whose structure depends upon their relations:

1.  **1:n (one-to-many) relationships**: *a `Shelf` has many `Book`s*.
2.  [**n:n (many-to-many) relationships**: *a `Topic` has and belongs to many `Book`s*.][crud-nn]
3.  **multiple named relationships**: *a `User` has many `Book`s they've created,*
    *and has and belongs to many favorite `Book`s they've "favorited."*
4.  **self referential relationships**:
    - ***(1:n)*** *a `User` has many `User`s as* ***followers***, and
    - ***(n:n)*** *a `User` has and belongs to many `User`s as* ***friends***.

## One-to-many (*1:n*)

As we know, the standard (ideal) relationship between two entities in
a relational structure is **1:n** (one-to-many). It's everywhere! Most of
the relationships we are going to deal with are straightforward 1:n. A
simple example is:

![ERD][erd-link]

…that is, in the language of Rails, **a shelf `has_many` books, and a book
`belongs_to` a shelf**. Rails uses the terms (and methods) `has_many` and
`belongs_to` to describe the two sides of this relationship.

**Note: there is one, big exception to the general rules given below!**
That exception is when two things are true:

- the independent, `has_many` entity is `User`, **and**
- the CRUDing logic states that the *current, logged-in user* is the only
  one who can relate to the dependent, `belongs_to` entity.

![ERD][erd-user-link]

When this is the case, you follow only *some* of the below directions:

- Migrations and Models: follow the steps exactly.
- Routes: **do not nest the resources** (skip this section)!
- Controllers: while the logic below holds for those routes, instead of
  finding the correct `@shelf` variable, replace it with the auth-created
  `current_user` method.
- View Templates & Forms: since you have not changed the routes, ignore
  this section!

Remember, this is only when the user that is relating to the dependent
entity is the *currently logged-in user*!

- **[Understanding and Context](#understanding-and-context)**
- **[Migrations](#migrations)**
- **[Models](#models)**
- **[Routes](#routes)**
- **[Controllers](#controllers)**
- **[View Templates & Forms](#view-templates--forms)**

### Understanding and Context

**In a `has_many` and `belongs_to` relationship there is a heirarchy.** The
`has_many` side (in this example, a **shelf**) controls the relationship.
Given this imbalance, it is always important to know which side of the
relationship you are on.

The `has_many` must exist *before* the `belongs_to` is created, and the 
`belongs_to` can't exist *after* the `has_many` has been destroyed. Thus 
there is an order for creation and destruction of instances of these 
resources, and also for definitions of them!

Thus, in this relationship, the `belongs_to` resource holds the data 
about the relationship. Ie, **one of the attributes of the `belongs_to`**
**(book) instance is its `has_many` (shelf)**, and not vice versa!

Also, very importantly, there is a difference when pluralizing related
resources! Remember, a book belongs to **a shelf**, but a shelf has 
many **books**.

Knowing the above, we can pretty much treat the `has_many` resource as
an **independent resource** that follows all of the rules for single model
CRUD. What becomes interesting is the `belongs_to`, or **dependent**
**resource**.

### Migrations

Migrations are straightforward. The independent, `has_many` resource is
created the same as usual:

```
$ rails g model Shelf category
```

The dependent, `belongs_to` resource is created after, and references
the first:

```
$ rails g model Book title body:text likes:integer shelf:references
```

**Note that the order is `resource_name:references`, and that the**
**resource name is singular.**

The output of the second generator should be (with some formatting
and additions):

```ruby
class CreateBooks < ActiveRecord::Migration
  def change
    create_table :books do |t|
      t.string     :title
      t.text       :body
      t.integer    :likes, default: 0
      t.references :shelf,  index: true, foreign_key: true

      t.timestamps null: false
    end
  end
end
```

### Models

If you generate the `belongs_to` resource as above, with a `references`
attribute, then the following will already by written. Check anyway:

**`/app/models/book.rb`:**

```ruby
class Book < ActiveRecord::Base
  belongs_to :shelf
end
```

You will *definitely* have to add the following to the `has_many`
resource:

**`/app/models/shelf.rb`:**

```ruby
class Shelf < ActiveRecord::Base
  has_many :books
end
```

**Note that the above are _methods_ that take _symbols_ as**
**arguments, and that the pluralization is different!**

Another thing to think about is whether or not you want to
enforce a "cascading delete" on the dependent resource. For
example, do you want to remove all of a given shelf's books
if that shelf is destoryed? You can do this simply by
adding to the model's definition:

```ruby
class Shelf < ActiveRecord::Base
  has_many :books, dependent: :destroy
end
```

To find out more, read the Rails Guides about [`has_many`][rails-has-many]
and [`belongs_to`][rails-belongs-to].

### Routes

As was described above, the `has_many` resource is independent. Thus
there needn't be any change to its routes.

**`/config/routes.rb`:**

```ruby
  resources :shelves
```

However, we can't fully CRUD the dependent `belongs_to` resource 
without knowing about the resource to which it belongs…
There is a pretty simple logic to it: sometimes we need to know
which shelf it is! Remember, **all of the information you need to
fulfill a request must be in that request**! HTTP has no context,
and the server doesn't know who anyone is!

We could do this with the simple setup:

```ruby
  resources :shelves
  resources :books
```

…but only by "hacking" the routes. For example, whenever we go to create
a book (or list them by shelf) we add a param to represent the shelf, like:

```
POST /books?shelf_id=1
GET  /books?shelf_id=1
```

But *Rails don't hack, son*. Instead, we can use something called
"nested routes" to make sure that we are creating routes that have all
of the necessary data structured into them *and* are (mostly) RESTful.

```ruby
  resources :shelves do
    resources :books
  end
```

This creates a series of routes that look like (from `rake routes`, and
excluding the `Shelf` routes, since those are the same):

```
         Prefix Verb   URI Pattern                                 Controller#Action
    shelf_books GET    /shelves/:shelf_id/books(.:format)          books#index
                POST   /shelves/:shelf_id/books(.:format)          books#create
 new_shelf_book GET    /shelves/:shelf_id/books/new(.:format)      books#new
edit_shelf_book GET    /shelves/:shelf_id/books/:id/edit(.:format) books#edit
     shelf_book GET    /shelves/:shelf_id/books/:id(.:format)      books#show
                PATCH  /shelves/:shelf_id/books/:id(.:format)      books#update
                PUT    /shelves/:shelf_id/books/:id(.:format)      books#update
                DELETE /shelves/:shelf_id/books/:id(.:format)      books#destroy
```

Thus, our new requests (to create or list a shelf's books) will look like:

```
POST /shelves/1/books
GET  /shelves/1/books
```

Great, right!? There's one more wrinkle, however. In our example, we 
need to know the shelf, at the very least, when the book is created. 
We may also want to know the shelf when we index (or list) the books 
in order to list all of a given shelf's books.

But, any route that includes the book's "ID" (show, edit, update,
and destroy) don't really *need* the shelf's "ID", since that's
implicit in the book, and can be found from ActiveRecord with
the simple methods:

```ruby
@book.shelf_id
#=> 1
@book.shelf
#=> <Shelf id:1>
```

We can thus clean up our routes by updating our `/config/routes.rb`
file to look like:

```ruby
  resources :shelves, shallow: true do
    resources :books
  end
```

We also may like to list *all* of our books, for all shelves! In
that case we want two index routes, one to list a shelf's books,
and one to list *all* books. The final setup looks like:

```ruby
  resources :books, only: [:index]
  resources :shelves, shallow: true do
    resources :books
  end
```

…which gives us the `rake routes`:

```
        Prefix Verb   URI Pattern                            Controller#Action
         books GET    /books(.:format)                       books#index
   shelf_books GET    /shelves/:shelf_id/books(.:format)     books#index
               POST   /shelves/:shelf_id/books(.:format)     books#create
new_shelf_book GET    /shelves/:shelf_id/books/new(.:format) books#new
     edit_book GET    /books/:id/edit(.:format)              books#edit
          book GET    /books/:id(.:format)                   books#show
               PATCH  /books/:id(.:format)                   books#update
               PUT    /books/:id(.:format)                   books#update
               DELETE /books/:id(.:format)                   books#destroy
```

**Note, as always, to only create the routes your appliction needs!**
**Don't create all of these routes, just use it as a template for the**
**routes you are going to define!**

To explore nested routes further, [check out the Rails Guides][rails-routes].

### Controllers

As we've seen a few times, the `has_many` resource is independent. Thus
there needn't be any change to its controller.

The differences begin again with the `belongs_to` controller, especially
for the routes we nested above: new, create and index.

The default actions would look like this, before changes:

**`/app/controllers/books_controller.rb`:**

```ruby
  def new
    @book = Book.new # this is passed to the form_for
  end
  
  def create
    @book = Book.new(book_params)
    
    if @book.save # validate the book
      flash[:notice] = "You've created a new book!"
      redirect_to book_path(@book)
    else
      render :new
    end
  end
  
  def index
    @books = Books.all
  end
```

This is no longer, sufficient, of course! We can break the necessary
changes down by action.

#### books#new

The route has been changed from `GET /books/new` to 
`GET /shelves/:shelf_id/books/new`. We need to load the shelf's
object in order to build the form correctly. Thus the new action looks
like:

```ruby
  def new
    @shelf = Shelf.find params[:shelf_id]
    @book = Book.new
  end
```

#### books#create

```ruby
  def create
    @shelf = Shelf.find params[:shelf_id]
    @book = shelf.books.build book_params
    
    if @book.save
      flash[:notice] = "You've created a new book!"
      redirect_to book_path(@book)
    else
      render :new
    end
  end
```

#### books#index 

So, while the basic index route (all books) is fine as it is,
we now need to check if we are handling that route or the new,
nested route `GET /shelves/:shelf_id/books`, and only show that
shelf's books.

```ruby
  def index
    if params[:shelf_id]
      @shelf = Shelf.find params[:shelf_id] 
      @books = @shelf.books
    else
      @books = Book.all
    end
  end
```

### View Templates & Forms

#### Path Helpers

When we begin using nested routes, the related path helpers change.
For the above book routes index (for a given shelf), create, and
new we now need to pass the relevant **shelf** object to them:

```ruby
books_path
#=> "/books"
shelf_books_path(@shelf)
#=> "/shelves/1/books"
new_shelf_book(@shelf)
#=> "/shelves/1/books/new"
```

#### Forms

As we've seen a few times, the `has_many` resource is independent. Thus
there needn't be any change to its view templates (except to add or remove 
whatever you want to be in the view!).

There is a change for the form_for helpers for the dependent resource,
however. This is because the routes that are generated (as an action)
for the forms are different!

The new `form_for` looks like:

**`/app/views/books/new.html.erb`** or **`/app/views/books/_form.html.erb`:**

```html
  <%= form_for [@shelf, @book] do |f| %>
    # …
```

<!-- LINKS -->

[crud-1n]: /crud_related_1n.md
[crud-nn]: /crud_related_nn.md
[crud-mu]: /crud_related_multiple.md
[crud-se]: /crud_related_self.md

[erd-link]:         /assets/img-crud-related-1n.jpg
[erd-user-link]:    /assets/img-crud-related-1n-user.jpg

[rails-routes]:     http://guides.rubyonrails.org/routing.html#nested-resources
[rails-has-many]:   http://guides.rubyonrails.org/association_basics.html#the-has-many-association
[rails-belongs-to]: http://guides.rubyonrails.org/association_basics.html#the-belongs-to-association


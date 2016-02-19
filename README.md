# CRUDing Related Models

While we went in to CRUDing in depth with a single model, and we explored
what it means to link, or **relate**, our models, we haven't shown you an
example of some of the tools Rails gives us to CRUD related models.

The [Rails Guides for ActiveRecord Associations][rails-guide] has great 
information on how to do this! You need to read them! But they don't walk
you through each step of implementing them in an app.

To that end, we have four examples/walkthroughs to show you about
how to structure the models, views, controllers and routes when trying
to CRUD resources whose structure depends upon their relations:

1.  [**1:n (one-to-many) relationships**: *a `Shelf` has many `Book`s*.][crud-1n]
2.  [**n:n (many-to-many) relationships**: *a `Topic` has and belongs to*
    *many `Book`s*.][crud-nn]
3.  [**multiple named relationships**: *a `User` has many `Book`s they've created,*
    *and habtm `Book`s they've "favorited."*][crud-mu]
4.  **self referential relationships**:
    - [***(1:n)*** *a `User` has many `User`s as* ***followers***][crud-s1], and
    - [***(n:n)*** *a `User` has and belongs to many `User`s as* ***friends***][crud-sn].

<!-- LINKS -->

[crud-1n]: /dist/crud_related_1n.md
[crud-nn]: /dist/crud_related_nn.md
[crud-mu]: /dist/crud_related_multiple.md
[crud-s1]: /dist/crud_related_self_1n.md
[crud-sn]: /dist/crud_related_self_nn.md
[rails-guide]: http://guides.rubyonrails.org/association_basics.html

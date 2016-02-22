# CRUDing Related Models — Self References, n:n


change back to underscore header.
<%= include_partial("_header.md", {s_n: true}) %>

## Self References: a User habtm Users as friends

Unlike following, friendship is symmetrical, therefore creating a
has_many_and_belongs_to relationship within one model. You may think we can
simply use a habtm relationship to itself:

![wrong ERD][erd-self-ref-idea-nn]

But, of course, that makes a nightmare of methods and terribly inefficient
database queries.

As with the previous self-referencing relationship, we'll of course use a join
table. However, unlike the `1:n`, we'll need two attachments to the same model.
This is due to the symmetrical nature of friendships.

![correct ERD][erd-self-ref-nn]

You'll have to specify the name of the join table, as well as the joining
columns just like the `1:n` self-referential relationship. In addition, we'll
need to create a flip-flop of the index, as we may need to locate friend's
friends in our site as well.

- **[Understanding and Context](#understanding-and-context)**
- **[Migrations](#migrations)**
- **[Models](#models)**
- **[Routes](#routes)**

### Understanding and Context

...

### Migrations

...

### Models

...

### Routes

...

<!-- LINKS -->

<%= include_partial("_links.md") %>

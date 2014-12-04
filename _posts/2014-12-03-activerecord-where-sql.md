---
layout: post
title: "ActiveRecord's #where & SQL"
identifier: activerecord-where-sql
tags: [blog, programming, web, ruby, Rails, ActiveRecord, associations, SQL, book, chapter, H.P. Lovecraft, Cthulhu]
image:
  feature: activerecord-where-sql/background.jpg
  credit:
  creditlink:
---

[ActiveRecord](https://github.com/rails/rails/tree/master/activerecord) is an
extremely powerful tool, allowing us as developers to be more productive by
writing less and doing more. Most of the time this *less* works to our
advantage, but sometimes *less* is not *more*.

<!-- excerpt -->

- - -

Imagine you have the following models:

{% highlight sql %}
CREATE TABLE books (
  id BIGSERIAL PRIMARY KEY,
  title TEXT,
  author TEXT
);

CREATE TABLE chapters (
  id BIGSERIAL PRIMARY KEY,
  book_id BIGINT REFERENCES books(id) NOT NULL,
  title TEXT,
  rank INTEGER
);
{% endhighlight %}

{% highlight ruby %}
class Book < ActiveRecord::Base
  has_many :chapters
end
{% endhighlight %}

{% highlight ruby %}
class Chapter < ActiveRecord::Base
  belongs_to :book
end
{% endhighlight %}

Notice that we have a `Book` model which has *title* and *author* attributes,
as well as a `Chapter` model which references a `Book` (via *book_id*) and
*title* attributes.

Since ActiveRecord provides easy interfaces for querying from our database, we
can quickly find all `Books` by an author, title, or containing a `Chapter`.

{% highlight ruby %}
> Book.where(author: "H. P. Lovecraft")
=> #<ActiveRecord::Relation [#<Book id: 1, title: "Necronomicon", author: "H. P. Lovecraft">, ...]>

> Book.find_by("title = 'Necronomicon'")
=> #<Book id: 1, title: "Necronomicon", author: "H. P. Lovecraft">

> chapter = Chapter.find_by(title: "The Call of Cthulhu")
=> #<Chapter id: 19, book_id: 1, title: "The Call of Cthulhu", rank: 19>
> chapter.book
=> #<Book id: 1, title: "Necronomicon", author: "H. P. Lovecraft">
{% endhighlight %}

If you were paying close attention to the above statements, you may have noticed
that two of the arguments were a hash, but one used a string. ActiveRecord
allows developers to pass a SQL string to be executed, among other things. For
the most part, passing a string into ActiveRecord's querying utilities will work
as expected...*for the most part*.

- - -

Once we get into more complex queries, ActiveRecord may help us avoid problems
that we're not always weary of. Notice how both of our `Book` and `Chapter`
models have a `title` attribute, what would happen if we tried to `join` these
two tables together while querying based on `title`?

{% highlight ruby %}
> Book.joins(:chapters).find_by("title = 'The Call of Cthulhu'")
PG::AmbiguousColumn: ERROR:  column reference "title" is ambiguous
LINE 1: ...ters" ON "chapters"."book_id" = "books"."id" WHERE (title = '...
{% endhighlight %}

Ah, an error! It looks like our database did not know which table's `title`
attribute we were interested in using. Now lets try the same method, but use a
hash instead.

{% highlight ruby %}
> Book.joins(:chapters).find_by(title: "The Call of Cthulhu")
=> nil
{% endhighlight %}

Nil? Well, at least this didn't raise any exceptions. To better understand what
happened, let's take a look at the generated SQL.

{% highlight sql %}
SELECT "books".* FROM "books" INNER JOIN "chapters" ON "chapters"."book_id" = "books"."id" WHERE ("books"."title" = 'The Call of Cthulhu') LIMIT 1
{% endhighlight %}

Near the end of the query, we see that ActiveRecord *automagically* selected our
`Book` model to be queried upon. This is very helpful of Rails, but we didn't
want to query by our `Books` title, but on our `Chapters` title. Enter nested
hashes.

{% highlight ruby %}
> Book.joins(:chapters).find_by(chapters: {title: "The Call of Cthulhu"})
=> #<Book id: 1, title: "Necronomicon", author: "H. P. Lovecraft">
{% endhighlight %}

Perfect! Now if we take a look at the generated SQL, we'll see that ActiveRecord
correctly queried by our `Chapters` table.

{% highlight sql %}
SELECT "books".* FROM "books" INNER JOIN "chapters" ON "chapters"."book_id" = "books"."id" WHERE ("chapters"."title" = 'The Call of Cthulhu') LIMIT 1
{% endhighlight %}

Although this won't fix all problems with ActiveRecord, *knowing is half the
battle*.

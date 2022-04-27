+++
title = "JSON — the relational database’s built-in ORM?"
date = 2020-11-20
[taxonomies]
categories = ["Programming"]
tags = ["SQL", "JSON", "Orm"]
+++

_originally posted on [Knowitlabs](https://knowitlabs.no/json-the-relational-databases-built-in-orm-965bd0905f4d)_

Database code. Queries, updates, and deletes. Repositories. Like many other developers, I have struggled with these things that tend to end up rather ugly. Not only might there be a lot of programming language (SQL) embedded inside another programming language (your app’s code), but there are a lot of mappings back and forth for data types, integration code, boilerplate! Or, you might be one of the lucky ones who get to use an Object Relational Mapping-library, clean, simple, and elegant code, right, hiding all of the interesting details? This post is about the quest for the Right Abstraction.

I’ve used two different relational databases professionally, first SQLite, and then PostgreSQL. I don’t consider myself very experienced in writing database code, but I do think that working with databases is one of the more enjoyable parts of being a developer. Except for that damn mapping code. In my first job, I used an in-house-developed ORM for SQLite, that needed constant modifications to be able to cope with the ever-growing complexity of queries coming from the layers above. ORM development tends to start out very simple: _right now I just need to read this or that table as a list_. But soon we need foreign keys and querying for trees, and before you know it you’ve reimplemented all the features of SQL itself. Just a lot worse, and what a mess you’re now in.

I _like_ working with different languages, they exist for a reason, I believe that SQL was designed to be written by humans instead of being a compiler target. I want to use all the cool features from my specific database implementation, and a general-purpose db-agnostic ORM might not support everything.

So I prefer writing out my SQL statements, but what about the “mapping” code?

Let’s start with a simple problem that might seem unrelated at first, with the usual boring tables:

```sql
CREATE TABLE customer
    id UUID PRIMARY KEY NOT NULL,
    name TEXT;CREATE TABLE order
    id UUID PRIMARY KEY NOT NULL,
    customer_id UUID NOT NULL REFERENCES customer (id),
    description TEXT;
```

The task at hand is to query for all customers and all orders for each customer (yes, I want it to be structured). There are several ways to do that. Often, to keep things very simple, the solution might be to issue [N+1](https://stackoverflow.com/questions/97197/what-is-the-n1-selects-problem-in-orm-object-relational-mapping) separate queries:

```sql
SELECT id, name FROM customer;
```

followed by

```sql
SELECT id, description
FROM order
WHERE customer_id = $customer_id;
```

issued for each of those initial rows. But this usually leads to performance issues because of missed optimization opportunities by the RDBMS, so I think we can do better.

Let’s leave that thought for a while and go back to the problem of deserializing rows into something that’s nice to work within my programming language of choice. For a _customer_, we usually want some kind of object having the fields `id` and `name`. A mapper, in pseudocode, could look like:

```javascript
function row_to_customer(row) {
    Customer(id = row['id'], name = row['name'])
}
```

That’s the type of boilerplate I don’t like writing. But it’s deserialization, and there’s one serialization format that any programming language will be able to deserialize for you (with hopefully very little boilerplate), and that’s _JSON_. These days I’m mostly using PostgreSQL and it has good support for JSON, and others have too. Let’s rewrite our query:

```sql
SELECT
    json_build_object(
        'id', id,
        'name', name
    )
FROM customer;
```

And just throw each row at our JSON deserializer, then there’s (almost) no mapping code to write anymore.

Our initial case with a list of orders for each customer, JSON solves that quite easily as well:

```sql
SELECT
    json_build_object(
        'id', id,
        'name', name,
        'orders', (
             SELECT
                 json_agg(
                     json_build_object(
                         'id', id
                         'description', description
                     )
                     ORDER BY description
                 )
             FROM order
             WHERE order.customer_id = customer.id
        )
    )
FROM customer;
```

`json_agg` will produce a nested list inside the outer object containing the rows of the subquery, and we’re done, without any additional mapping code, except the language-specific hints potentially needed to deserialize this into the object:

```java
class CustomerWithOrders {
    id: UUID,
    name: String,
    orders: Array<Order>
}
```

I’ve used the JSON/SQL technique with great success lately, especially for queries that involve querying many tables where a single big, fat old JOIN won’t cut it. It’s very fast: The database is allowed to do its own optimizations, and JSON is usually extremely fast to process by the application.

Could we try a similar thing for insertion?

```sql
INSERT INTO order (customer_id, description)
    VALUES (
       ($1->>'customer_id')::UUID,
       $1->>'description'
    );
```

Now you only have to bind one parameter instead of two and avoid at least one level of Repeating Yourself.

Is it fair to call this the _database’s built-in ORM_? We used json_build_**object**, right? Seriously, I’m not sure, but it was a fun thought anyway.

I hope you learned a useful programming pattern, at least it took a while for me to discover it.

+++
title = 'No More Unchecked Sqlx Queries'
date = 2024-09-02T22:20:00+02:00
+++

This post is intended for SQLx users, who want to take advantage of the powerful compile time checks SQLx crate provides. More specifically, we look into practical ways how to make any SQLx query checked.

The Rust compiler can catch many mistakes, preventing bugs from happening at runtime. This makes Rust a very robust language to work with. If I compare working with Rust with any of my previous projects, I do see less regressions in general.

As this high level of robustness comes from compile time checks, Rust feels comparatively a bit more brittle at the edges. There may be limitations to compiler checks, when dealing with external data sources like databases and JSON from third party APIs.

With this post, I hope to offer a useful reference, for how to avoid writing unchecked queries in the first place, and of course, how to refactor existing unchecked queries into checked ones. Different flavours of unchecked and checked queries alike are presented, so that the reader can see the practical differences, and keep refining their craft.

First we look into what are the differences between a checked and an unchecked queries, and the reasons to prefer checked queries. Then we cover two cases where we might feel tempted to write an unchecked query, but fortunately, won't have to. Most examples are written for Postgres, but can be applied to other relational databases as well. I’ve added one example for MySQL in that vein.

## What are checked SQLx queries?

[SQLx](https://docs.rs/sqlx/latest/sqlx/index.html) is a popular and full-fledged crate for interacting with relational databases. It supports PostgreSQL, MySQL and SQLite. While there are several other crates available, SQLx is a pretty safe pick, unless you want support for [ORMs](https://en.wikipedia.org/wiki/Object%E2%80%93relational_mapping), which SQLx does not provide. In that case you may want to look into [diesel](https://docs.rs/diesel/latest/diesel/).

One of my favorite things about SQLx are the compile-time checks, because they really help to make the application more robust. We just need to use them!

## Checked query vs. unchecked query

A checked query is a query that can be statically validated at compile time. SQLx compares column and table names to the database schema, validates the syntax, and verifies data types without running any code.

An unchecked query has no guarantees at all. It could contain any number of typos and issues with types, and the compiler cannot catch them.

I think unchecked queries are a bad practice, to put in bluntly. With unchecked queries, we give up all guarantees on the correctness of the query. Also, as this blog post aims to show, they can be avoided altogether.

## Why care about unchecked queries?

If you have a test DB, and run all the queries against it, fine, your test suite will catch the issue. But what if you don't? Then, checked queries will save your day. And even if you do, they'll save you some time anyway, because you'll get corrective feedback already at compile time, just as we Rust developers like it.

After all, an unchecked SQL query is just a string, from the compiler's perspective. The syntax could be invalid, or there could be a typo in the column name, and the compiler could not catch it.

The power of Rust lies in compiler checks (or at least a decent share), so let's keep our queries checked!

## How to recognize unchecked queries in the wild?

As a general rule, queries created using the [sqlx::query()](https://docs.rs/sqlx/latest/sqlx/fn.query.html) and [sqlx::query_as()](https://docs.rs/sqlx/latest/sqlx/fn.query_as.html) functions are unchecked, while those created with the macros [sqlx::query!](https://docs.rs/sqlx/latest/sqlx/macro.query.html) and [sqlx::query_as!](https://docs.rs/sqlx/latest/sqlx/macro.query_as.html) (and several others) are checked. If you see [QueryBuilder](https://docs.rs/sqlx/latest/sqlx/struct.QueryBuilder.html) being used, that query is not checked. Check the documentation for other functions and macros.

Personally I like to use [sqlx::query_as!](https://docs.rs/sqlx/latest/sqlx/fn.query_as.html) macro with an appropriate struct, except maybe for very simple queries, when I would use the [sqlx::query!()](https://docs.rs/sqlx/latest/sqlx/macro.query.html) macro. In our project, we mainly use these two, and steer clear from functions that don't check the query.

## Temptations

The idea of checked queries, is that if SQLx is not happy with the query, the compiler will complain. There are some specific cases however, where it may not be evident, how to construct that checked query.

After analyzing our codebase, I've identified two cases where one may be tempted to write an unchecked query. First is the case of different variants of essentially the same query. The second case involves the usage of IN operator. In this post we review both.

## Dealing with variants of the same query

One may feel tempted to write unchecked queries, when there are subtle variants of the same query.

To give some practical examples, let's imagine a basic table called `users`, with the following fields.

```sql
+----------------+-------------+
| Column Name    | Data Type   |
+----------------+-------------+
| id             | UUID        |
| name           | VARCHAR     |
| email          | VARCHAR     |
| created_at     | TIMESTAMP   |
| updated_at     | TIMESTAMP   |
| is_guest       | BOOLEAN     |
+----------------+-------------+
```

Why would we ever have several variants of essentially the same query?

Let's imagine the following use cases:
- Fetch all users, or only those who are guest users, or only those who have already registered (i.e. are no longer guest users)
- Fetch all users, who were last updated before a specific date 
- Fetch all users, who were last updated between two dates
- Fetch a user or several, with a specific list of ids

Not sure if it is typical just for gaming, where data models do change and sometimes we need to patch live data, but we do have variants of quite a few queries in the codebase.

Perhaps we need to update or fetch data for existing users, based on when they were last updated. Perhaps we know that users updated after a specific date will already have the correct data. Hence the queries may have different parameters, to get exactly the correct results.

Additionally, these queries may be quite large, and indeed you may want to write the query just once, even if it has variations.

One may be tempted to write unchecked queries just for convenience. Let's take a brief look at what not to do.

## Quick and dirty SQL string

The quick and dirty approach is to write the SQL query as a separate &str. Maybe like this

```rust
let mut query = "SELECT id FROM users"; // Imagine a more complex query here

// There are three variants, depending on the value of is_guest_option
if let Some(is_guest) = is_guest_option {
    query = format!("{query} WHERE is_guest = {is_guest}");
}

let res = sqlx::query(query).fetch_all(&pool).await; // Unchecked query
```

The resulting query is unchecked, because this query is built dynamically. Hence there are no guarantees on its correctness.

## Parameterized query

A parameterized query seems more evolved, and curiously enough some [LLMs](https://en.wikipedia.org/wiki/Large_language_model) recommend this pattern.

```rust
let mut query = "SELECT id FROM users".to_string(); // Imagine a more complex query here

if let Some(is_guest) = is_guest_option {
    query.push_str(" WHERE is_guest = ?");
}

let res = sqlx::query(&query) // Unchecked query
    .bind(is_guest) // Bind the parameter dynamically
    .fetch_all(&pool)
    .await;
```

Binding parameters feels kind of smart too. However, I would not recommend this approach, as the query is still unchecked.

## How to make these queries checked?

This is the heart of the matter, how to make these queries checked. Conditional compilation is one alternative, and using an optional parameter another. Let's review these alternatives.

## Conditional compilation works, but is repetitive

```rust
// This is checked, but not DRY
let query = if let Some(is_guest) = is_guest_option {
    sqlx::query!("SELECT id FROM users WHERE is_guest = ?", is_guest)
} else {
    sqlx::query!("SELECT id FROM users")
};

let res = query
    .fetch_all(&pool)
    .await;
```

This query is checked, so it is definitely better than sprinkling the codebase with unchecked queries. However, in each of the branches we need to write the entire query, making the code repetitive and harder to maintain. Managing more complex queries, especially with many variants, will become quite unwieldy.

What would be a better alternative? Well, we can use what I like to call "Option pattern". It will be easier to manage, especially when the queries grow in size.

## Option pattern to the rescue

Let's take our good old `SELECT id FROM users` query. Let's also have more variation, so we can see firsthand how nicely we can compose queries with the Option pattern. Specifically, we will add optional date parameters to the mix, that allow us to search for users that were last updated before a date, or after it, or without date restrictions. We also add the option to limit the query to just guest users, or only to the registered users (not guests), if that parameter is passed.

```rust
// Postgres version
let ids = sqlx::query_as!(
    Uuid,
    "SELECT id FROM users \
     WHERE ($1::timestamptz IS NULL OR updated_at < $1) \
       AND ($2::timestamptz IS NULL OR updated_at > $2) \
       AND ($3::boolean IS NULL OR is_guest = $3)",
    updated_before_option,
    updated_after_option,
    is_guest_option,
    )
    .fetch_all(&pool)
    .await;
```
 
Pretty neat! A checked query, that supports different variants. As `None` evaluates to `NULL`, we conveniently omit the condition for that parameter, when the `Option` is `None`. Postgres requires explicit type casting in this case, which is why we use the `::` type cast operator, as seen above.

The same idea should work with MySQL as well, adjusting the syntax slightly. MySQL uses ? as parameter placeholder markers, so the query would look like so. The casts are implicit, unlike in Postgres.

```sql
// MySQL version
let ids = sqlx::query_as!(
    Uuid,
    "SELECT id FROM users \
     WHERE (? IS NULL OR updated_at < ?) \
       AND (? IS NULL OR updated_at > ?) \
       AND (? IS NULL OR is_guest = ?)",
    updated_before_option,
    updated_before_option,
    updated_after_option,
    updated_after_option,
    is_guest_option,
    is_guest_option,
    )
    .fetch_all(&pool)
    .await;
```
I've come across one more situation where we may be tempted to write an unchecked query. 

## When IN does not compile, use ANY instead

SQLx has an important limitation when using the [IN operator](https://www.postgresql.org/docs/current/functions-subquery.html#FUNCTIONS-SUBQUERY-IN) with list parameters. The `IN` operator works with a single item, subqueries, or hard-coded values. However, it cannot be included in a checked query, if it takes a list as a parameter. This can be confusing when encountered for the first time, so it is useful to mention in this post as well.

The solution is to use the [ANY operator](https://www.postgresql.org/docs/current/functions-subquery.html#FUNCTIONS-SUBQUERY-ANY-SOME), which works seamlessly with lists and individual items alike.

Here's an example of a checked query using `IN` that will not compile in:

```rust
// Does not compile
let records = sqlx::query!("SELECT name, email, created_at \
    FROM users WHERE id IN ($1)",
    ids,
    )
    .fetch_all(&pool)
    .await;`
```

To address this issue, we can use `ANY` operator instead:

```rust
// This compiles
let records = sqlx::query!("SELECT name, email, created_at \
    FROM users WHERE id = ANY($1)",
    ids,
    )
    .fetch_all(&pool)
    .await;
```

And there, we have a checked query! Here are some [further reflections](https://stackoverflow.com/questions/34627026/in-vs-any-operator-in-postgresql/34627688#34627688) on the rather subtle difference between IN and ANY.

## Final thoughts

I believe that with these two tools in our toolkit, there should be no need to write an unchecked SQLx query, ever.

Let me know if you have a use case for unchecked queries that cannot be effectively covered with the ideas from this blog post. I'm more than happy to change my mind when new information arises. Hope it has been a useful read. Until next time!

# Basic Query Builder

# Why

1. Simple, lightweight, and fast
2. Supports any and all syntax by the nature of how it works
3. Doesn't require learning special syntax or operators
4. &gt;90% test coverage

# Examples

## Basic

```golang
q := bqb.New("SELECT * FROM places WHERE id = ?", 1234)
sql, params, err := q.ToSql()
```

Produces
```sql
SELECT * FROM places WHERE id = ?
```
```
PARAMS: [1234]
```

## Postgres

Just call the `ToPsql()` method instead of `ToSql()` to convert the query to Postgres syntax

```golang
q := bqb.New("DELETE FROM users").
    Space("WHERE id = ? OR name IN (?)", 7, []string{"delete", "remove"}).
    Space("LIMIT ?", 5)
sql, params, err := q.ToPsql()
```

Produces
```sql
DELETE FROM users WHERE id = $1 OR name IN ($2, $3) LIMIT $4
```
```
PARAMS: [7, "delete", "remove", 5]
```

## Query Building

Since queries are built in an additive way by reference rather than value, it's easy to mutate a query without
having to reassign the result.

### Basic Example

```golang
sel := bqb.New("SELECT")

...

// later
sel.Space("id")

...

// even later
sel.Comma("age").Comma("email")
```

Produces
```sql
SELECT id,age,email
```

### Advanced Example

The `Empty()` function returns a query that resolves to an empty string if no query parts have
been added via methods on the query instance. For example `q := Empty("SELECT")` will resolve to
an empty string unless parts have been added by one of the methods,
e.g `q.Space("* FROM my_table")` would make `q.ToSql()` resolve to `SELECT * FROM my_table`.

```golang

sel := bqb.Empty("SELECT")

if getName {
    sel.Comma("name")
}

if getId {
    sel.Comma("id")
}

if !getName && !getId {
    sel.Comma("*")
}

from := bqb.Empty("FROM")
from.Space("my_table")

where := bqb.Empty("WHERE")

if filterAdult {
    adultCond := bqb.Empty().
        And("name = ?", "adult")
    if ageCheck {
        adultCond.And("age > ?", 20)
    }
    where.And("(?)", adultCond)
}

if filterChild {
    where.Or("(name = ? AND age < ?)", "youth", 21)
}

q := bqb.New("? ? ? LIMIT ?", sel, from, where, 10)
```

Assuming all values are true, the query would look like:
```sql
SELECT name,id FROM my_table WHERE (name = 'adult' AND age > 20) OR (name = 'youth' AND age < 21) LIMIT 10
```

If `getName` and `getId` are false, the query would be
```sql
SELECT * FROM my_table WHERE (name = 'adult' AND age > 20) OR (name = 'youth' AND age < 21) LIMIT 10
```

If `filterAdult` is `false`, the query would be:
```sql
SELECT name,id FROM my_table WHERE (name = 'youth' AND age < 21) LIMIT 10
```

If all values are `false`, the query would be:
```sql
SELECT * FROM my_table LIMIT 10
```

# Frequently Asked Questions

## Is there more documentation?

It's not really necessary because the API is so tiny.
Most of the documentation will be around how to use SQL.

## Why not just use a string builder?

Bqb provides several benefits over a string builder:

For example let's say we use the string builder way to build the following:

```golang
var params []interface{}
q := "SELECT * FROM my_table WHERE "
if filterAge {
    params = append(params, 21)
    q += fmt.Sprintf("age > $%d ", len(params))
}

if filterBobs {
    params = append(params, "Bob%")
    q += fmt.Sprintf("name LIKE $%d ", len(params))
}

if limit != nil {
    params = append(params, limit)
    q += fmt.Sprintf("LIMIT $%d", len(params))
}

// SELECT * FROM my_table WHERE age > $1 AND name LIKE $2 LIMIT $3
```

Some problems with that approach
1. If `filterBobs` and `filterAge` are false, the query has an empty where clause
2. You must remember to include a trailing space for each clause
3. You have to keep track of parameter count (for Postgres anyway)
4. It's kind of ugly

The same logic can be achieved with `bqb` a bit more cleanly

```golang
q := bqb.New("SELECT * FROM my_table")
where := bqb.Empty("WHERE")
if filterAge {
    where.And("age > ?", 21)
}

if filterBobs {
    where.And("name LIKE ?", "Bob%")
}

q.Space("?", where)

if limit != nil {
    q.Space("LIMIT ?", limit)
}

// SELECT * FROM my_table WHERE age > $1 AND name LIKE $2 LIMIT $3
```

Both methods will allow you to remain close to the SQL, however the `bqb` approach will
1. Easily adapt to MySQL or Postgres without changing parameters
2. Hide the "WHERE" clause if both `filterBobs` and `filterAge` are false


## Why not use a full query builder?

Take the following _typical_ query example:

```golang
q := qb.Select("*").From("users").Where(qb.And{qb.Eq{"name": "Ed"}, qb.Gt{"age": 21})
```

Vs the bqb way:

```golang
q := bqb.New("SELECT * FROM users WHERE name = ? AND age > ?", "ed", 21)
```

## Okay, so a simple query it might make sense to use something like `bqb`, but what about grouped queries?


A query builder can handle this in multiple ways, a fairly common pattern might be:

```golang
q := qb.Select("name").From("users")

and := qb.And{}

if checkAge {
    and = append(and, qb.Gt{"age": 21})
}

if checkName {
    or := qb.Or{qb.Eq{"name":"trusted"}}
    if nullNameOkay {
        or = append(or, qb.Is{"name": nil})
    }
    and = append(and, or)
}

q = q.Where(and)

// SELECT name FROM users WHERE age > 21 AND (name = 'trusted' OR name IS NULL)
```

Contrast that with the `bqb` approach:

```golang

q := bqb.New("SELECT name FROM users")

where := bqb.Empty("WHERE")

if checkAge {
    where.And("age > ?", 21)
}

if checkName {
    or := bqb.New("name = ?", "trusted")
    if nullNameOkay {
        or.Or("name IS ?", nil)
    }
    where.And("(?)", or)
}

q.Space("?", where)

// SELECT name FROM users WHERE age > 21 AND (name = 'trusted' OR name IS NULL)
```

It seems to be a matter of taste as to which method appears cleaner.





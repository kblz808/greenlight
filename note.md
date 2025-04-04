When PostgreSQL is freshly installed it only has one user account: a superuser called 
`postgres`.


During installation, an operating system user named postgres should also have been 
created on your machine.

`SELECT current_user` query to see which PostgreSQL user you currently are.

`psql -U postgres`
> connect to postgres with the username postgres

`CREATE DATABASE greenlight;`
> create a new database called greenlight

`\c greenlight;`
> connect to the greenlight database

the `\` character indicates a meta command

`\d {table_name}`
> view table fields

`\l`
> list all databases

`\dt`
> list tables

`\du`
> list users

`CREATE ROLE greenlighti WITH LOGIN PASSWORD 'pa55word';`
> create a new role called greenlight

--- 6

## migrations
For every change that you want to make to your database schema
(like creating a table, adding a column, or removing an unused index)
you create a pair of migration files. One file is the ‘up’ migration
which contains the SQL statements necessary to implement the change, and
the other is a ‘down’ migration which contains the SQL statements to
reverse (or roll-back) the change

Each pair of migration files is numbered sequentially, usually 0001, 0002,
0003... or with a Unix timestamp, to indicate the order in which migrations
should be applied to a database.

Using migrations to manage your database schema, rather than manually
executing the SQL statements yourself, has a few benefits:
+ The database schema (along with its evolution and changes) is 
completely described by the ‘up’ and ‘down’ SQL migration files.
And because these are just regular files containing some SQL statements, 
they can be included and tracked alongside the rest of your code in a version control system.

+ It’s possible to replicate the current database schema precisely on
another machine by running the necessary ‘up’ migrations

To manage SQL migrations in this project we’re going to use the
migrate command-line tool 

The first thing we need to do is generate a pair of migration files using
the migrate create command

`$ migrate create -seq -ext=.sql -dir=./migrations create_movies_table`
+ `-seq` indicates that we want to use sequential numbering like 0001, 0002, for
the migration files
+ `-ext` indicates that we want to give the migration file the extension `.sql`

`migrate -path=./migrations -database=$GREENLIGHT_DB_DSN up`
> executing the migration

if you want to see which migration version your database is
currently on you can run the migrate tool’s version command.
`migrate -path=./migrations -databse=$GREENLIGHT_DB_DSN version`

we can also migrate up or down to a specific version by using the goto
command
`migrate -path=./migrations -database=$EXAMPLE_DSN goto 1`

You can use the down command to roll-back by a specific number of migrations. For
example, to rollback the most recent migration you would run
` migrate -path=./migrations -database =$EXAMPLE_DSN down 1`

When you run a migration that contains an error, all SQL
statements up to the erroneous one will be applied and then the
migrate tool will exit with a message describing the error.

--- 9

```sql
SELECT id, created_at, title, year, runtime, genres, version
FROM movies
WHERE (LOWER(title) = LOWER($1) OR $1 = '')
AND (genres @> $2 OR $2 = '{}')
ORDER BY id
```

This SQL query is designed so that each of the filters behaves like it is ‘optional’.

the condition `(LOWER(title) = LOWER($1) OR $1 = '')` will evaluate as true if
the placeholder parameter $1 is a case-insensitive match for the movie title or the
placeholder parameter equals ''. So this filter condition will essentially be ‘skipped’ when
movie title being searched for is the empty string ""

The `(genres @> $2 OR $2 = '{}')` condition works in the same way. The @> symbol is the
‘contains’ operator for PostgreSQL arrays, and this condition will return true if each value
in the placeholder parameter $2 appears in the database genres field or the placeholder
parameter contains an empty array

```sql
SELECT id, created_at, title, year, runtime, genres, version
FROM movies
WHERE (to_tsvector('simple', title) @@ plainto_tsquery('simple', $1) OR $1 = '')
AND (genres @> $2 OR $2 = '{}')
ORDER BY id
```

The `to_tsvector('simple', title)` function takes a movie title and splits it into lexemes.
We specify the simple configuration, which means that the lexemes are just lowercase
versions of the words in the title . For example, the movie title "The Breakfast Club"
would be split into the lexemes 'breakfast' 'club' 'the'

The `plainto_tsquery('simple', $1)` function takes a search value and turns it into a
formatted query term that PostgreSQL full-text search can understand.
It normalizes the search value (again using the simple configuration), strips any special characters, and
inserts the and operator & between the words. As an example, the search value "The Club"
would result in the query term 'the' & 'club'

The `@@` operator is the matches operator. In our statement we are using it to check whether
the generated query term matchesthe lexemes. To continue the example, the query term
'the' & 'club' will match rows which contain both lexemes 'the' and 'club'

==

In our case it makes sense to create `GIN` indexes on both the genres field and the lexemes
generated by `to_tsvector()`, both which are used in the WHERE clause of our SQL query
`migrate create -seq -ext .sql -dir ./migrations add_movies_indexes`

`migrate -path ./migrations -database $GREENLIGHT_DB_DSN up`

==

we want to let the client control the sort order via a query
string parameter in the format `sort={-}{field_name}`

Behind the scenes we will want to translate this into an `ORDER` BY clause in our SQL query,
so that a query string parameter like sort=-year would result in a SQL query like this:

```sql
SELECT id, created_at, title, year, runtime, genres, version
FROM movies
WHERE (STRPOS(LOWER(title), LOWER($1)) > 0 OR $1 = '')
AND (genres @> $2 OR $2 = '{}')
ORDER BY year DESC --<-- Order the result by descending year
```

The difficulty here is that the values for the ORDER BY clause 
will need to be generated at runtime based on the query string 
values from the client.

we’ll need to interpolate these dynamic values into our query using
`fmt.Sprintf()`

it’s also important to be aware that the order of returned
rows is only guaranteed by the rulesthat your `ORDER BY` clause imposes

Within our application, we’ll just need to translate the `page` and `page_size` values provided
by the client to the appropriate `LIMIT` and `OFFSET` values for our SQL query

```sql
LIMIT = page_size
OFFSET = (page - 1) * page_size
```

==

The inclusion of the `count(*)` `OVER()` expression at the start of the query will result in the
filtered record count being included as the first value in each row

--- 12

|signal|description|shortcut|catchable|
|---|---|---|---|
|SIGINT|Interrupt from keyboard|Ctrl+C|Yes|
|SIGQUIT|Quit from keyboard|Ctrl+\|Yes|
|SIGKILL|Kill process (terminate immediately)|-|No|
|SIGTERM|Terminate process in orderly manner|-|Yes|

`pgrep -l api`
> search for a running process called api

`pkill -SIGKILL api`
> send the SIGKILL signal to the api process

`pkill -SIGTERM api`
> send the SIGTERM signal to the api process

--- 15

`migrate create -seq -ext .sql -dir ./migrations create_tokens_table`

```sql
CREATE TABLE IF NOT EXISTS tokens (
  hash bytea PRIMARY KEY,
  user_id bigint NOT NULL REFERENCES users ON DELETE CASCADE,
  expiry timestamp(0) with time zone NOT NULL,
  scope text NOT NULL
);
```

We also use the `ON DELETE CASCADE` syntax to instruct PostgreSQL to automatically
delete all recordsfor a user in our `tokens` table when the parent record in the users table
is deleted.

==

```sql
SELECT users.id, users.created_at, users.name, users.email, users.password_hash, users.activated, users.version
FROM users
INNER JOIN tokens
ON users.id = tokens.user_id
WHERE tokens.hash = $1
AND tokens.scope = $2
AND tokens.expiry > $3
```

we’re using the `ON users.id = tokens.user_id` clause to
indicate that we want to join records where the user id value equalsthe token `user_id`

--- 16

Authentication is about confirming who a user is, whereas authorization is
about checking whether that user is permitted to do something

==

If no Authorization header was provided at all, then we will add the details for an
anonymous user to the request context instead.

we'll be storing the user detailsin the request context.

+ Every http.Request that our apprication processes has a `context.Context` embedded in
it, which we can use to store key/varue pairs containing arbitrary data during the lifetime
of the request. In this case we want to store a User struct containing the current user’s
information

+ Any values stored in the request context have the type any. This means that after
retrieving a value from the request context you need to assert it back to its original type
before using it.

--- 17

The relationship between permissions and users is a great example of a many-to-many
relationship. One user may have many permissions, and the same permission may belong
to  many users.

The classic way to manage a many-to-many relationship in a relational database like
PostgreSQL is to create a joining table between the two entities.

Then we can create a joining table called users_permissions to store the information about
which users have which permissions.

```
PermissionModel.GetAllForUser(user) → Retrieve all permissions for a user
UserModel.GetAllForPermission(permission) → Retrieve all users with a specific permission
```

the sql migrations: `migrate create -seq -ext .sql -dir ./migrations add_permissions`

--- 18

if two URLs have the same scheme, host and port (if specified) they are said to
share the same origin.

|A|B|Same origin?|Reason|
|---|---|---|---|
|https://foo.com/a|http://foo.com/a|No|Different shcme (http vs https)|
|http://foo.com/a|http://www.foo.com/a|No|Different host (foo.com vs www.foo.com)|
|http://foo.com/a|http://foo.com:443/a|No|Different port (no port vs 443)|
|http://foo.com/a|http://foo.com/b/|Yes|Only the path is different|
|http://foo.com/a|http://foo.com/a?b=c|Yes|Only the query string is different|
|http://foo.com/a#b|http://foo.com/a#c|Yes|Only the fragment is different|

all web browsers implement a
security mechanism known as the same-origin policy. 

+ A webpage on one origin can embed certain types of resources from another origin in
  their HTML — including images, CSS, and JavaScript files.
> `<img src="http://anotherorigin.com/example.png" alt="example image">`

+ A webpage on one origin can send data to a different origin. For example, it’s OK for a
  HTML form in a webpage to submit data to a different origin.

+ But a webpage on one origin is not allowed to receive data from a different origin.

the same-origin policy prevents a (potentially
malicious) website on another origin from reading (possibly confidential) information from
your website.

let’s say that you have a webpage at https://foo.com containing some frontend JavaScript code. If this JavaScript tries to make an HTTP request to
https://bar.com/data.json (a different origin), then the request will be sent and
processed by the bar.com server, but the user’s web browser will block the response so that
the JavaScript code from https://foo.com cannot see it.

if you have an API at api.example.com and a trusted JavaScript front-end
application running on www.example.com, then you’ll probably want to allow cross-origin
requests from the trusted www.example.com domain to your API.

he Access-Control-Allow-Origin response header is used to indicate to a browser that
it’s OK to share a response with a different origin. 
`Access-Control-Allow-Origini: *`

==

`go run ./cmd/api -cors-trusted-origins="http://localhost:9000 http://localhost:9001"`

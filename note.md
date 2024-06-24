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

---

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

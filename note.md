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

`\l`
> list all databases

`\dt`
> list tables

`\du`
> list users

`CREATE ROLE greenlighti WITH LOGIN PASSWORD 'pa55word';`
> create a new role called greenlight

the `bin` direcotry will contain our compiled application binaries

the `cmd/api` directory will contain the application-specific code for the api
this will include the code for running the server, reading and writing HTTP requests, and managing auth

the `internal` directory will contain various packages used by our api
it will contain the code for interacting with our database, doing data valdation, 
sending emails and so on. (any code which isnt application specific and can be reused)

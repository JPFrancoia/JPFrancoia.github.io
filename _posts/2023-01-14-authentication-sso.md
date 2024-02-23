---
layout: posts
title:  "Authentication workflow with SSO"
image: authentication_list.jpg

excerpt: "This article will demonstrate how to register a user through a
Single Sign-On provider like Google or Facebook. The implementation will be
done in Go, with the Gin framework. We'll also see how to write new users
to a Postgres database."

---

# Authentication workflow with SSO

In this didactic article, we'll implement a basic authentication workflow
with a Single Sign-On (SSO) provider. I chose Facebook, but it should
not be difficult to port the code to Google or Github. We'll use the
information returned by the SSO provider to register a user into a database.
The website/app will be written in Go with the Gin framework.
To summarize we will:

- Register users with a SSO provider (no form!)
- Add users to our database


The final state of the code is available [here](https://github.com/JPFrancoia/authentication_example).

**NOTE**: this is a toy example to get familiar with authentication and JWT
tokens. I do not recommend using this setup in production. For production
applications, I would always recommend to not implement authentication
yourself, it's not worth your time / the risk. [Auth0](https://auth0.com/)
provides a generous free tier and a solid SDK that should allow you to get
started quickly and securely.


## Login and index pages

We will use two very simple HTML pages in this article:

- `index.html`: a simple index page
- `login.html`: the page that will allow users to login/register

Here is the content of `index.html`:

```html
<!DOCTYPE html>
<html>
    <head>
        <title>Simple Test</title>
    </head>
    <body>
        <h1>
            Hello, test!
        </h1>
    </body>
</html>
```

Here is the content of `login.html`:

```html
<!DOCTYPE html>
<html>
    <head>
        <title>Login</title>
    </head>
    <body>
        <a href="/auth/facebook">Login with Facebook</a>
    </body>
</html>
```

## Building the backend

**NOTE**: This article isn't about Go per se so I'll be brief on the Go aspect. You can
find out more about Go [here](https://go.dev/doc/). If you're not interested in
the basic setup of the app, skip this section.

I will use the following structure for this project:

```
.
├── api
│   ├── data_registy
│   │   └── data_registry.go
│   ├── entities
│   │   └── entities.go
│   ├── handlers
│   │   └── auth
│   │       └── auth.go
│   ├── main.go
│   └── templates
│       ├── index.html
│       └── login.html
├── docker-compose.yml
├── go.mod
├── go.sum
└── README.md
```

Don't worry too much if you don't understand yet what every file is doing,
I'll detail them one by one in this article. At the moment we only need to
focus on the `main.go` file and the `templates/` directory. `main.go` is the
entry point for the app, and `templates/` contains the two pages mentioned in
the section above.

In its initial form, `main.go` is very simple:

```go
package main

import (
	"fmt"
	"net/http"

	"github.com/gin-gonic/gin"
)

func health(c *gin.Context) {
	c.JSON(http.StatusOK, "healthy")
}

func index(c *gin.Context) {
	c.HTML(http.StatusOK, "index.html", gin.H{})
}

func login(c *gin.Context) {
	c.HTML(http.StatusOK, "login.html", gin.H{})
}

func main() {
	fmt.Println("Starting")

	router := gin.Default()

	router.LoadHTMLGlob("templates/*")

	router.GET("/health", health)

	router.GET("/", index)
	router.GET("/login", login)

	router.Run("localhost:8080")
}
```

As it is, `main.go` creates a Gin app, loads the HTML templates and creates
three *endpoints*:

- `/` -> the index page
- `/login` -> the login page
- `/health` -> I'll explain in a bit what it does

To start the application, you can `cd` to the `api` directory and run:

```
go run .
```

You can also use [air](https://github.com/cosmtrek/air): it does exactly
the same, but it also watches changes on the files in your directory and
automatically reloads the app.

You should see something like that:

```
Starting
[GIN-debug] [WARNING] Running in "debug" mode. Switch to "release" mode in production.
 - using env:   export GIN_MODE=release
 - using code:  gin.SetMode(gin.ReleaseMode)

[GIN-debug] Loaded HTML Templates (3): 
        - 
        - index.html
        - login.html

[GIN-debug] GET    /health                   --> main.health (1 handlers)
[GIN-debug] GET    /                         --> main.index (1 handlers)
[GIN-debug] GET    /login                    --> main.login (1 handlers)
[GIN-debug] [WARNING] You trusted all proxies, this is NOT safe. We recommend you to set a value.
Please check https://pkg.go.dev/github.com/gin-gonic/gin#readme-don-t-trust-all-proxies for details.
[GIN-debug] Listening and serving HTTP on localhost:8080
```

If you now head to `http://localhost:8080/` in your browser you should see "Hello, test!".
You can also use `curl` to call an endpoint from the command line:

```
❯ curl "http://localhost:8080/health"
"healthy"%
```

At this point you should have a runnable application, without any
authentication mechanism. In the second part of the article we'll implement
the authentication flow to register and login our users.

### The `health` endpoint

This endpoint isn't strictly necessary for a simple project, however I think
it's a good idea to always implement it. The idea behind it is simple: if
the endpoint answers with a 200 HTTP code, the app is healthy. If not (if
it's a 500 for example), the app isn't healthy. *Something* can call this
endpoint and can decide to restart the app if it isn't healthy. Kubernetes
for example uses this mechanism to make sure an app is always healthy. A more
useful version of this endpoint would be:

```go
func health(c *gin.Context) {

	// Make sure the database is reachable
	err := data_registry.PingDB()
	if err != nil {
		c.JSON(http.StatusInternalServerError, "unhealthy")
		return
	}

	c.JSON(http.StatusOK, "healthy")
}
```

---

### Facebook SSO

We will use Facebook as the Single Sign-On (SSO) provider. We're using a SSO
provider because it provides a good user experience, and it delegates the
identity check to Facebook. In a nutshell, here is how it works:

- from your app, the user clicks a link that redirects them to Facebook
- Facebook asks the user if they want to give some permissions to *your* app
    (e.g: access to their name, email, etc)
- if the user agrees, Facebook calls a predetermined endpoint of your app and
    attaches the user's information to the call
- your app processes the user's information and creates a user record in the
    database

For the first step, you app's backend will have to authenticate
with Facebook.  You'll have to register your app with Facebook and
obtain a *client ID* and a *client secret*. The general steps are
described below but you can find more information [in the official
documentation](https://developers.facebook.com/docs/development/create-an-app/).

- first, you'll need to create an app. Head to [https://developers.facebook.com/apps/
    ](https://developers.facebook.com/apps/)
- click on the "**Create app**" button and follow the guided steps until your app is created
- choose "**Consumer**" for the type of the app
- head to "**Settings > Basic**" on the left pane, And write down the App Id and
    the App secret (we'll call them client ID and client secret)
- next, add the product called "**Facebook login**", and choose the "**web**" option in the quickstart selection.
- When Facebook asks you to fill the URL of your website, enter `localhost:8080`

### Authentication flow

Let's get coding! We can now add this function in `handlers/auth/auth.go`:

```go
package auth

import (
	"fmt"
	"net/http"
	"os"

	data_registry "local/auth_example/api/data_registy"
	ent "local/auth_example/api/entities"

	"github.com/gin-gonic/gin"
	"github.com/markbates/goth"
	"github.com/markbates/goth/gothic"
	"github.com/markbates/goth/providers/facebook"
)

// Initialize the Facebook provider for the auth flow
func init() {
	facebookProvider := facebook.New(
		os.Getenv("FACEBOOK_CLIENT_ID"),
		os.Getenv("FACEBOOK_CLIENT_SECRET"),
		os.Getenv("AUTH_REDIRECT_URL"),
	)
	goth.UseProviders(facebookProvider)
}

func Login(c *gin.Context) {
	// Insert provider into context
	// https://github.com/markbates/goth/issues/411#issuecomment-891037676
	q := c.Request.URL.Query()

	q.Add("provider", c.Param("provider"))
	c.Request.URL.RawQuery = q.Encode()

	fmt.Println("Starting auth flow")
	gothic.BeginAuthHandler(c.Writer, c.Request)
}
```

You will notice that the `init()` function is taking three parameters from
the environment. This is a good development practice, because this kind
of variables are likely to change between deployments (like dev, staging
or production) and it allows us to decouple the code from the config, in
accordance to the [Twelve-Factor App principles](https://12factor.net/).
You can set these variables for your session by running the commands
below, or you can use a `.envrc` file (I also recommend having a look at
[direnv](https://direnv.net/)).

```
export FACEBOOK_CLIENT_ID=YOUR_CLIENT_ID
export FACEBOOK_CLIENT_SECRET=YOUR_CLIENT_SECRET
export AUTH_REDIRECT_URL=http://localhost:8080/auth/callback
```

The new iteration of `main.go` looks like this:

```go
func main() {
	router := gin.Default()

	router.LoadHTMLGlob("templates/*")

	router.GET("/health", health)

	router.GET("/", index)
	router.GET("/login", login)

	authGroup := router.Group("/auth")
	{
		authGroup.GET("/:provider", auth.Login)
		authGroup.GET("/callback", auth.AuthCallback)
	}

	router.Run("localhost:8080")
}
```

Notice this line:

```go
authGroup.GET("/:provider", auth.Login)
```

This translates to: *"call the `auth.Login` function with the `provider`
parameter, and take this parameter from the URL"*. So when the user clicks
this link (from `login.html`), we'll start the authentication process with
the Facebook provider.

```html
<a href="/auth/facebook">Login with Facebook</a>
```

When Facebook is done setting permissions, it will call our `/auth/callback`
endpoint. Here is the function and the data structures that will take care
of the query:

```go
import (
	"time"

	"github.com/google/uuid"
	"github.com/markbates/goth"
)

func AuthCallback(c *gin.Context) {
	user, err := gothic.CompleteUserAuth(c.Writer, c.Request)

	if err != nil {
		c.AbortWithError(http.StatusUnauthorized, err)
		return
	}

	ourUser := ent.GothUser(user).ToUser()

	c.Redirect(http.StatusFound, "/")
}

type User struct {
	UserId       uuid.UUID `db:"user_id" json:"user_id"`
	CreationTime time.Time `db:"creation_time" json:"creation_time"`
	Provider     string    `db:"provider" json:"provider"`
	Email        string    `db:"email" json:"email"`
}

// Alias the non-local goth.User type so that I can implement the ToUser()
// method for it
type GothUser goth.User

func (gothUser GothUser) ToUser() User {
	return User{
		UserId:       uuid.New(),
		CreationTime: time.Now(),
		Provider:     gothUser.Provider,
		Email:        gothUser.Email,
	}
}
```

At this point the `AuthCallback` function produces a `ourUser`
data structure that we can write to the database. I'm used to put
the data structures in the `entities` module (because I like the [clean
architecture](https://jpfrancoia.github.io/2023/03/20/clean-architecture-python.html)),
but you can keep them in `auth.go` if you want.


### Postgres with docker-compose

We will create a local Postgres database to store our users. These days Docker
(and docker-compose) is the simplest way to spin up a database locally. To
create a Postgres database, we can use the `docker-compose.yml` file below:

```yaml
version: "3.9"
services:
  pin_db:
    container_name: auth_db
    image: postgres:14
    restart: unless-stopped
    environment:
      POSTGRES_HOST_AUTH_METHOD: "trust"
      POSTGRES_DB: auth_example
      POSTGRES_USER: postgres
    ports:
      - 5432:5432
    healthcheck:
      test: pg_isready -U postgres -d auth_example
      interval: 10s
      timeout: 3s
      retries: 3
    mem_limit: 4g
    shm_size: 1g
```

With these settings, we'll create a database called `auth_example` and a
`postgres` user. The connection to the database will not require a password
(this is a local database, do NOT do that with a production database). The
settings also implement a healthcheck, which will probe the database to
make sure it's ready to accept connections.

You can `cd` in the same directory as the file and run this command:

```
docker compose up -d
```

And you can then check that the database is ready with:

```
> docker ps

CONTAINER ID   IMAGE                    COMMAND                  CREATED         STATUS                            PORTS                                                  NAMES
5e3c78494ac4   postgres:14              "docker-entrypoint.s…"   4 seconds ago   Up 3 seconds (health: starting)   0.0.0.0:5432->5432/tcp, :::5433->5432/tcp              auth_db
```

### Database migrations for the `users` table

Before we can write anything into the database, we'll
need to create the `users` table in the database. We could
do that manually, but it's a better practice to use [schemas
migrations](https://en.wikipedia.org/wiki/Schema_migration). The idea
is simple: every time you need to change the schema of your database, you
create a migration `up` and a migration `down`. The `up` migration will
bring the schema to its new state, and the `down` will revert the changes
(useful if anything goes wrong). To manage these migrations, I like to use
[dbmate](https://github.com/amacneil/dbmate). It's a super simple tool,
language agnostic, and it's not coupled with any ORM (as opposed to tools
like SQLAlchemy). Let's create our migrations:

```shell
dbmate new users_table
```

This will create a file `db/migrations/SOME_TIMESTAMP_users_table.sql`. Let's
fill it in:

```sql
-- migrate:up

create extension if not exists "uuid-ossp";

create type provider_type as enum (
    'facebook'
);

create table users (
    user_id uuid not null default uuid_generate_v4 (),
    creation_time timestamp with time zone not null default now(),
    provider provider_type not null,
    email text not null,
    primary key (user_id),
    unique (email, provider)
);

create index idx_hash_user_id on users using hash (user_id);

-- migrate:down
drop index idx_hash_user_id;

drop table users;

drop type provider_type;
```

Let's detail these queries:

- We'll use `create extension` to allow the database to create UUIds
- We create an enum for the provider: there is only one possible provider,
    `facebook`, this field can't take another value. No need for this field to
    be `text`
- every combination of email and provider should be unique (to avoid duplicate
    users)
- We create a `hash` index on the `user_id` field (as opposed to a BTREE
    index). Here this is me just being fancy, a classic index works well too,
    and I haven't validated the benefits of using a hash index

`dbmate` needs an env variable to be set to be able to reach the database, so
run the following command (or use a `.envrc` file):

```
export DATABASE_URL="postgres://postgres@localhost:5433/auth_example?sslmode=disable"
```

We can now create the table by running:

```shell
❯ dbmate up
Applying: 20230316175233_users_table.sql
Writing: ./db/schema.sql
```

Congratulations! we now have a `users` table in the database:

```
+---------------+--------------------------+--------------------------------------+
| Column        | Type                     | Modifiers                            |
|---------------+--------------------------+--------------------------------------|
| user_id       | uuid                     |  not null default uuid_generate_v4() |
| creation_time | timestamp with time zone |  not null default now()              |
| provider      | provider_type            |  not null                            |
| email         | text                     |  not null                            |
+---------------+--------------------------+--------------------------------------+
Indexes:
    "users_pkey" PRIMARY KEY, btree (user_id)
    "users_email_provider_key" UNIQUE CONSTRAINT, btree (email, provider)
    "idx_hash_user_id" hash (user_id)
```


### Data registry

I like using an abstraction layer to interact with databases, and most
of the time I call it the **data registry**. It's responsible for dealing
with the database. Please note that it's not a reimplementation of an
ORM, it's just an abstraction layer that decouples the application from
any database logic. The rest of the application sends *entities* (like our
`User` struct above) to the data registry, and the registry takes care of
writing the data into the database. I'll detail why I think it's a good
design choice in an article dedicated to the clean architecture.


```go
package data_registry

import (
	"fmt"

	ent "local/auth_example/api/entities"

	"github.com/jmoiron/sqlx"
	_ "github.com/lib/pq"
)

var db sqlx.DB

// Open the DB connection and cache the query strings.
// Should be called once when the app starts.
func InitDB(connStr string) error {

	db_con, err := sqlx.Open("postgres", connStr)

	db = *db_con

	if err != nil {
		return err
	}

	if err := db.Ping(); err != nil {
		return err
	}

	return nil
}

// Make sure the DB is accessible.
// Return an error if it's not
func PingDB() error {
	if err := db.Ping(); err != nil {
		return err
	}

	return nil
}

// Upsert a user into the DB
func UpsertUser(user ent.User) error {

	fmt.Println(user)

	_, err := db.NamedExec(
		"insert into users (user_id, creation_time, provider, email) values (:user_id, :creation_time, :provider, :email) on conflict (email, provider) do nothing;",
		user,
	)

	if err != nil {
		fmt.Println(err)
		return err
	}

	return nil
}
```

The code above initializes a database connection and defines two simple
functions:
- `PingDB`: make sure the registry can access the database. We can use this
    function in the app's health endpoint
- `UpsertUser`: insert a new User into the database, if the user doesn't
    already exist in the database

We can now complete the `AuthCallback` function that we saw earlier:

```go
func AuthCallback(c *gin.Context) {
	user, err := gothic.CompleteUserAuth(c.Writer, c.Request)

	if err != nil {
		c.AbortWithError(http.StatusUnauthorized, err)
		return
	}

	ourUser := ent.GothUser(user).ToUser()

	// Write the new user to database.
	err = data_registry.UpsertUser(ourUser)
	if err != nil {
		c.AbortWithError(http.StatusInternalServerError, err)
		return
	}

	c.Redirect(http.StatusFound, "/")
}
```

But let's not forget to initialize the data registry in `main.go`:

```go
func main() {

	// Crash early if connection to DB fails
	if err := data_registry.InitDB(os.Getenv("DATABASE_URL")); err != nil {
		log.Fatal("Failed to open a database connection: ", err)
	}
...
}
```

For the code above to work, we will need to set another env variable:

```
export DATABASE_URL="postgres://postgres@localhost:5432/auth_example?sslmode=disable"
```

## Running the app

We can finally `cd` into the `api` directory and run the code!

```
❯ go run .
[GIN-debug] [WARNING] Creating an Engine instance with the Logger and Recovery middleware already attached.

[GIN-debug] [WARNING] Running in "debug" mode. Switch to "release" mode in production.
 - using env:   export GIN_MODE=release
 - using code:  gin.SetMode(gin.ReleaseMode)

[GIN-debug] Loaded HTML Templates (3): 
        - 
        - index.html
        - login.html

[GIN-debug] GET    /health                   --> main.health (3 handlers)
[GIN-debug] GET    /                         --> main.index (3 handlers)
[GIN-debug] GET    /login                    --> main.login (3 handlers)
[GIN-debug] GET    /auth/:provider           --> local/auth_example/api/handlers/auth.Login (3 handlers)
[GIN-debug] GET    /auth/callback            --> local/auth_example/api/handlers/auth.AuthCallback (3 handlers)
[GIN-debug] [WARNING] You trusted all proxies, this is NOT safe. We recommend you to set a value.
Please check https://pkg.go.dev/github.com/gin-gonic/gin#readme-don-t-trust-all-proxies for details.
[GIN-debug] Listening and serving HTTP on localhost:8080
Starting auth flow
[GIN] 2023/03/17 - 21:43:40 | 307 |    1.196956ms |       127.0.0.1 | GET      "/auth/facebook"
[GIN] 2023/03/17 - 21:43:41 | 302 |  486.132495ms |       127.0.0.1 | GET      "/auth/callback?code=AQBKIiXXXXXXXXXXXXXXXXXXXXXXX"
[GIN] 2023/03/17 - 21:43:41 | 200 |      88.006µs |       127.0.0.1 | GET      "/"
```

The console output above is what you should see if you run the app in your
console and if you click the "Login with facebook" link in your browser.

Looking into the database, we can see that a new user is now registered:

```
+--------------------------------------+-------------------------------+----------+--------------------------------+
| user_id                              | creation_time                 | provider | email                          |
|--------------------------------------+-------------------------------+----------+--------------------------------|
| b42bf029-4423-43e9-b178-cb8972baa4ac | 2023-03-17 21:38:25.950247+00 | facebook | jeanpatrick.francoia@gmail.com |
+--------------------------------------+-------------------------------+----------+--------------------------------+
```

## Conclusion

In this article, we saw how to create an authentication mechanism with
SSO end to end. This will allow you to register/login users into your app
without making them fill a form and create yet another password. But this
is just a first step, because we haven't seen the *authorization* part. In
a future article we'll see how we can create JWT tokens for our users, which
will allow us to validate the identity of a user throughout their session on
your app. We will also be able to restrict some pages to unregistered users.

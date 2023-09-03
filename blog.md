# Creating a full-stack Rust & Vue app

This is actually my first attempt at documenting my thoughts as I approach a new project. So, not only will I be learning the best way to write things down as I do them, but also how to setup a Rust backend using Axum and SQLx in Docker. What could go wrong?

For anyone following along, this post does assume that you have `npm` and `Rust` already installed. I'm performing everything on an M1 MacOS Pro, so some things may differ depending on your machine OS and architecture. 

## Helpful VSCode Extensions
These are some of the VSCode extensions that I frequently use and recommend for this project:
- [crates](https://marketplace.visualstudio.com/items?itemName=serayuzgur.crates)
- [Docker](https://marketplace.visualstudio.com/items?itemName=ms-azuretools.vscode-docker)
- [Error Lens](https://marketplace.visualstudio.com/items?itemName=usernamehw.errorlens)
- [rust-analyzer](https://marketplace.visualstudio.com/items?itemName=rust-lang.rust-analyzer)
- [TypeScript Vue Plugin (Volar)](https://marketplace.visualstudio.com/items?itemName=Vue.vscode-typescript-vue-plugin)
- [Vue Language Features (Volar)](https://marketplace.visualstudio.com/items?itemName=Vue.volar)

## Project Creation
To create the project, we'll go step by step to see how to scaffold everything and eventually fill it out. 

### Frontend
We start off easy enough by creating the project skeleton. I create the rust-fullstack-template folder and run `npm init vue@latest` to initialize the project creation wizard. Walking through the questions, I create a TypeScript-based Vue frontend with Pinia. 

Opening some of the frontend files to change the tab spacing from 2 to 4 (the correct amount of spaces), I'm presented with a bunch of angry, red, squiggle lines. My flawless run is at end. Searching for my error

`Cannot find module 'vue-router' or its corresponding type declarations.`

quickly leads me to a solution.  A Stack Overflow post links to [the Vue documentation](https://vuejs.org/guide/typescript/overview.html#volar-takeover-mode) that solves my problem. I'm still not 100% clear on WHY it was happening, but the errors seems to be largely erroneous since it builds and runs perfectly fine.

Next, we're off to install Tailwind CSS, a utility-first CSS framework. Fortunately for us, [Tailwind has all the documentation we need](https://tailwindcss.com/docs/guides/vite#vue) to get setup in our Vue project. A quick check of `npm run dev` to make sure our project runs and it looks like we're ready for the next step. 

### Backend
First, we run `cargo new rust-fullstack-backend` to create the Rust project that will become our backend API. From the rust-fullstack-backend directory, we do a quick `cargo run` to verify that everything installed correctly. Seeing a "Hello, world!" in our console, it looks like we're good to go.

To start out, we'll need to add a few dependencies to our `Cargo.toml` file. The `crates` extension I mentioned at the top is really great here to keep track of what versions are available for a given crate.

`Cargo.toml`:
```
[dependencies]
axum = "0.6.19"
tokio = { version = "1.29.1", features = ["full"] }
tracing = "0.1.37"
tracing-subscriber = "0.3.17"
serde = { version = "1.0.171", features = ["derive"] }
serde_json = "1.0.103"
```

For creating the beginnings of the API, I referenced [this blog post by Carlos Marcano](https://carlosmv.hashnode.dev/getting-started-with-axum-rust) that I found. Since there's not much point in just repeating what I found in there, I'll be doing a lot of hand-waving for the code portion and just trying to focus on capturing my thoughts.

One thing I didn't see mentioned as I'm going through is the use of `tracing` and `tracing-subscriber`. Basically, we'll be using those for our async logging. Once we fill out the initial code for `main.rs`, we'll run 
```
cargo run
```
and we should see "listening on 127.0.0.1:3000" in the console. Going to `127.0.0.1:3000` in our browser and we see "Hello, World!". Success!

We're not really interested in the rest of the blog since the examples are pretty generic to `axum`. The one exception is the `json_hello` function since we'll be dealing with JSON responses at some point. It looks like the formatting got messed up on their blog, so I'll post it here. This function will go at the bottom of our `main.rs` file. 

```Rust
async fn json_hello(Path(name): Path<String>) -> impl IntoResponse {
    let hello = String::from("Hello ");
    let greeting = name.as_str();

    (StatusCode::OK, Json(json!({"message": hello + greeting })))

}
```

If you're not familiar with Rust, you might be confused with the return type being `impl IntoResponse`. All that means is that whatever we return from this function must implement the `IntoResponse` trait, the trait `axum` expects for endpoints. 

All we need to do now is add it to our routes in our `main` function:
```Rust
let app = Router::new()
    .route("/", get(root))
    .route("/hello/:name", get(json_hello));
```
This lets our app know that the `/hello` route takes a parameter, `name`, and sends a GET request to our `json_hello` function. If we enter `localhost:3000/hello/Michael`, we'll see:
```json
{"message":"Hello Michael"}
```
Now that we have the general scaffolding we need for our backend, we can start working on creating our database and containerizing our application with Docker.

### Docker
In our root directory, we start out with the `docker-compose.yml` file:
```yaml
version: "3.8"
services:
  db:
    container_name: rust-fullstack-db
    image: postgres:15.3
    ports:
      - "5432:5432"
    command: postgres -c max_connections=500
    environment:
      - POSTGRES_USER=${DB_USER}
      - POSTGRES_PASSWORD=${DB_PASSWORD}
    env_file:
      - ./.env
    volumes:
	  - ./db:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER}"]
      interval: 30s
      timeout: 30s
      retries: 5
    networks:
      - rust-fullstack-network

  backend:
    container_name: rust-fullstack-backend
    restart: always
    build:
      context: ./backend
    volumes:
      - ./rust-fullstack-backend:/code/app
    ports:
	  - 3000:3000
    env_file:
      - ./.env
    networks:
      - rust-fullstack-network
    command: cargo run

  frontend:
    container_name: rust-fullstack-frontend
    restart: always
    build:
      context: ./frontend
    volumes:
      - ./rust-fullstack-frontend:/app
      - /app/node_modules/
    ports:
	  - 5173:5173
    networks:
      - rust-fullstack-network
    command: npm run dev -- --host

networks:
  rust-fullstack-network:
  driver: bridge
  external: true
  name: rust-fullstack-network
```
With the exception of the `db` container, each container will use a Dockerfile that we specify to build the image.
`rust-fullstack-backend/Dockerfile`:
```Dockerfile
FROM rust:1.71
WORKDIR /code
COPY . /code/app/
WORKDIR /code/app
RUN cargo install --path .
```

`rust-fullstack-frontend/Dockerfile`:
```Dockerfile
FROM node:18-alpine
COPY ./ /app
WORKDIR /app
RUN npm install
```

Once we've created the necessary files, we can run
```bash
> docker network create rust-fullstack-network
> docker-compose up -d 
```

If we check, we should be able to access our endpoints at `localhost:3000` and the frontend app at `localhost:5173`. And... we can't access our endpoints. Turns out we need to update `main.rs`.

Change 
```rust
let addr = SocketAddr::from(([127, 0, 0, 1], 3000));
```
to
```rust
let addr = SocketAddr::from(([0, 0, 0, 0], 3000));
```
Looks like that fixed it! You can also test out the database by connecting via your GUI of choice (I'm currently trying out [Beekeeper Studio](https://github.com/beekeeper-studio/beekeeper-studio)). There's nothing there, but we're about to change that.

### Database
First, we'll make sure we add the necessary dependencies to our `Cargo.toml` file:
```toml
[dependencies]
...
anyhow = "1.0.72"
sqlx = { version = "0.7.1", features = ["runtime-tokio-native-tls", "json", "postgres"] }
```

Next, we need to add our model struct. We'll add a couple of files and a new directory in `rust-fullstack-backend/src/`. This will probably seem unnecessarily verbose for our simple app, but it's a good foundation if we want to build on it later. So, the new file structure will look like this:
```
src/
|--model/
|  |--user.rs
|--main.rs
|--model.rs
```
`model.rs` will allow us to call our struct defined in `user.rs` in `main.rs` and elsewhere.

`model.rs`:
```Rust
mod user
```

`model/user.rs` is where we define our simple `User` struct:
```Rust
use serde::{Deserialize, Serialize};

#[derive(sqlx::FromRow, Deserialize, Serialize)]
pub struct User {
	pub id: i32,
	pub first_name: String,
	pub last_name: String,
	pub email: String,
}
```

In order to run our migrations, we'll install `sqlx-cli`:
```Bash
> cargo install sqlx-cli
```

Now, we create our database and our first migration:
```Bash
> sqlx database create
> sqlx add user
```

Ok, it looks like it needs the `--database-url` parameter and our database URL. To save you some headache, I'll document what I did to solve this. I'm not sure why we need to do it this way since we can connect to the database through our SQL GUI of choice.

Steps to solve:
```Bash
> docker-compose up
> docker exec -it rust-fullstack-backend bash
> cargo install sqlx-cli
> sqlx database create --database-url postgresql://postgres:postgres@db:5432/rust_api
> sqlx migrate add user
```
The fourth step there will change depending on what you set your database username, password, and port to. I'm calling my database `rust_api`, but feel free to change that to whatever you want. Just remember to change it to yours whenever we reference it. We should probably add the install line to our Dockerfile as well so that we don't have to do it every time.

The migrate line creates our first migration file, which is useful for retaining the database structure. Inside the created `migrations/<timestamp>_user.sql` file, we'll add a command to create the `users` table:
```SQL
CREATE TABLE users (
	id SERIAL PRIMARY KEY,
	first_name VARCHAR(255) NOT NULL,
	last_name VARCHAR(255) NOT NULL,
	email VARCHAR(255) NOT NULL UNIQUE,
);
```

Generally, I prefer to stick to the standard of not having plural table names, but since `user` is a reserved keyword in Postgres, we'll just make do with `users`.

Save that and run the following in the backend container:
```Bash
> sqlx migrate run --database-url postgresql://postgres:postgres@db:5432/rust_api
```

If that was successful, then you should see the following response in the terminal:
```bash
Applied <timestamp>/migrate user
```

Next, it's back over to our backend `main.rs` file to make sure we can actually connect to the database. We'll start by adding `use sqlx::postgres::PgPoolOptions;` to our list of imports. Then, we'll need to update our `main` function to connect to the database. Once we do, it should look like this:
```Rust
mod model;
mod util;

#[tokio::main]
async fn main() -> Result<()> {
    let database_url = util::get_database_url();
    
    tracing_subscriber::fmt::init();

    let pool = PgPoolOptions::new()
        .max_connections(50)
        .connect(&database_url)
        .await
        .context("could not connect to database_url")?;

    let app = Router::new()
        .route("/", get(root))
        .route("/hello/:name", get(json_hello));

    let addr = SocketAddr::from(([0, 0, 0, 0], 3000));
    tracing::info!("listening on {}", addr);

    axum::Server::bind(&addr)
        .serve(app.into_make_service())
        .await
        .unwrap();

        Ok(())
}
```

The `util::get_database_url()` part is from a utility file that we'll create to help aggregate all of the functions like getting our database URL. We'll add the `util.rs` file at the same level as `main.rs`, just under `src/`. The function simply formats all of our Docker environment variables into a single `String` and returns it.

```Rust
use std::env;

pub fn get_database_url() -> String {
    let username = env::var("DB_USER").expect("DB_USER must be set");
    let password = env::var("DB_PASSWORD").expect("DB_PASSWORD must be set");
    let host = env::var("DB_HOST").expect("DB_HOST must be set");
    let db_name = env::var("DB_NAME").expect("DB_NAME must be set");

    format!(
        "postgres://{}:{}@{}/{}",
        username, password, host, db_name
    )
}
```

Each one has an `expect`, which will cause the program to panic and print the message if the environment variable doesn't exist.

For this next part, I'll be sticking the code into the `model/user.rs` file since it's code that interacts with the `User` model, but feel free to place it somewhere else if you prefer a bit more separation.  Add the following to the top of the file
```Rust
use axum::response::IntoResponse;
use axum::http::StatusCode;
use axum::{Extension, Json};

use sqlx::PgPool;
```
Then add the `all_users()` function
```Rust
pub async fn all_users(Extension(pool): Extension<PgPool>) -> impl IntoResponse {
	let sql = "SELECT * FROM users ".to_string();

	let user = sqlx::query_as::<_, User>(&sql)
		.fetch_all(&pool)
		.await
		.unwrap();

	(StatusCode::OK, Json(user))
}
```
The function is fairly straightforward and returns a JSON list of all of the users in the database. Based on what I was able to find, we use `axum`'s  `Extension` for our parameter type since it helps to share state across the app and the function needs access to the SQL pool. That means that later, if we forget to add a layer to our `main.rs` file with the SQL pool, our app will fail at runtime, preventing some awkward errors later when trying to interact with the API.

Before we get any further into building out our endpoints, let's add a couple of custom errors for them in `src/errors.rs`:
```Rust
use axum::{http::StatusCode, response::IntoResponse, Json};
use serde_json::json;


pub enum CustomError {
	BadRequest,
	UserNotFound,
	InternalServerError,
}

impl IntoResponse for CustomError {
	fn into_response(self) -> axum::response::Response {
		let (status, error_message) = match self {
			Self::InternalServerError => (
				StatusCode::INTERNAL_SERVER_ERROR,
				"Internal Server Error",
			),
			Self::BadRequest=> (StatusCode::BAD_REQUEST, "Bad Request"),
			Self::UserNotFound => (StatusCode::NOT_FOUND, "User Not Found"),
		};
		(status, Json(json!({"error": error_message}))).into_response()
	}
}
```
Remember to add `mod errors` to `main.rs` so that our program recognizes the new file. At the top of our `user.rs` file, we need to add the following line so that we can actually use our new custom errors:
```Rust
use crate::errors::CustomError;
```

\* Michael from the future here to list all of the imports so you don't see errors later:
```Rust
use axum::response::IntoResponse;
use axum::http::StatusCode;
use axum::{Extension, Json};
use axum::extract::Path;

use serde::{Deserialize, Serialize};
use serde_json::{json, Value};
use sqlx::PgPool;

use crate::errors::CustomError;
```

Within the same file, we'll add some more functions:
- retrieve a user by their ID (`GET`) 
- create a new one (`POST`)
- update a user (`PUT`)
- remove a user (`DELETE`)
To avoid a wall of code, I'll separate the blocks by function. The first is our function to retrieve a user by their ID:
```Rust
pub async fn user(Path(id):Path<i32>, Extension(pool): Extension<PgPool>) -> Result<Json<User>, CustomError> {
	let sql = "SELECT * FROM users WHERE id=$1".to_string();
	
	let user: User = sqlx::query_as(&sql)
		.bind(id)
		.fetch_one(&pool)
		.await
		.map_err(|_| {
			CustomError::UserNotFound
		})?;

	Ok(Json(user))
}
```
`Path` is a nice helper struct provider by `axum` that parses the URL and automatically decodes any percent encoded parameters (you've probably seen these in URLs where there's a "%20" instead of a space). `Extension` is one we're now familiar with, and we can see that the return type is either a JSON response of our User or our custom error. 

For a new user, some of the code is pretty similar, but we're now going to be including a `NewUser`  struct. For those who are used to Python ORMs, this should seem familiar. We're basically just defining what fields we want to include when creating a new user. Since the database automatically increments the `id` field, we just need to include the other three fields.
```Rust
#[derive(sqlx::FromRow, Deserialize, Serialize)]
pub struct NewUser {
	pub first_name: String,
	pub last_name: String,
	pub email: String,
}
...
pub async fn new_user(
	Json(user): Json<NewUser>,
	Extension(pool): Extension<PgPool>
) -> Result<(StatusCode, Json<NewUser>), CustomError> {
	if user.first_name.is_empty()|| user.last_name.is_empty() || user.email.is_empty() {
		return Err(CustomError::BadRequest)
	}
	let sql = "INSERT INTO users (first_name, last_name, email) VALUES ($1, $2, $3)";

	let _ = sqlx::query(&sql)
		.bind(&user.first_name)
		.bind(&user.last_name)
		.bind(&user.email)
		.execute(&pool)
		.await
		.map_err(|_| {
			CustomError::InternalServerError
		})?;

	Ok((StatusCode::CREATED, Json(user)))
}
```
We perform some basic validation to make sure that none of the fields are empty, then we insert the new user into the database.
```Rust
#[derive(sqlx::FromRow, Deserialize, Serialize)]
pub struct UpdateUser {
	pub first_name: String,
	pub last_name: String,
	pub email: String,
}
...
pub async fn update_user(
	Path(id): Path<i32>,
	Json(user): Json<UpdateUser>,
	Extension(pool): Extension<PgPool>
) -> Result<(StatusCode, Json<UpdateUser>), CustomError> {
	if user.first_name.is_empty() || user.last_name.is_empty() || user.email.is_empty() {
	return Err(CustomError::BadRequest)
}

	let sql = "SELECT * FROM user WHERE id=$1".to_string();

	let _user: User = sqlx::query_as(&sql)
		.bind(id)
		.fetch_one(&pool)
		.await
		.map_err(|_| {
			CustomError::UserNotFound
		})?;

	let _ = sqlx::query("UPDATE user SET first_name=$1, last_name=$2, email=$3 WHERE id=$4")
		.bind(&user.first_name)
		.bind(&user.last_name)
		.bind(&user.email)
		.bind(id)
		.execute(&pool)
		.await;

	Ok((StatusCode::OK, Json(user)))
}
```
In `update_user()`, you'll notice we're not actually using any of the variables that are being assigned (denoted solely by or prefixed with `_` ). That's because we only actually care about the success or failure of the query. The simple validation check helps guardrail the fields being empty and the first query helps guardrail against us updating a user that doesn't exist.

Similarly, we check if a user exists before attempting to delete them:
```Rust
pub async fn delete_user(
	Path(id): Path<i32>,
	Extension(pool): Extension<PgPool>
) -> Result<(StatusCode, Json<Value>), CustomError> {
	let _find: User = sqlx::query_as("SELECT * FROM user WHERE id=$1")
		.bind(id)
		.fetch_one(&pool)
		.await
		.map_err(|_| {
			CustomError::UserNotFound
		})?;

	sqlx::query("DELETE FROM user WHERE id=$1")
		.bind(id)
		.execute(&pool)
		.await
		.map_err(|_| {
			CustomError::UserNotFound
		})?;

	Ok((StatusCode::OK, Json(json!({"msg": "User Deleted"}))))
}
```

Great, we now have some basic CRUD functionality for our `User` model. Next, we just need to add our new functions to the router in `main.rs`.  Add our new dependencies at the top:
```Rust
use axum::{
	routing::{get, post, put, delete},
	http::StatusCode,
	response::IntoResponse,
	extract::Path,
	Json, Router, Extension
};
```
Then, update the routes:
```Rust
let app = Router::new()
	.route("/", get(root))
	.route("/hello/:name", get(json_hello))
	.route("/users", get(model::user::all_users))
	.route("/user", post(model::user::new_user))
	.route("/user/:id", get(model::user::get_user))
	.route("/user/:id", put(model::user::update_user))
	.route("/user/:id", delete(model::user::delete_user))
	.layer(Extension(pool));
```
Hmmm... There's something about our `new_user()`  and `update_user()` functions that the compiler doesn't like. 

\*Several Hours Later\*

Ok, so after digging around, trying to decipher the error message it was giving us, I came across `axum`'s  `#[debug_handler]`. It's a nice feature that gives us a more detailed error. Turns out, we just need to move the `Json` type parameters to be the last parameter in each of our functions. Here's the full error:
```Rust
`Json<_>` consumes the request body and thus must be the last argument to the handler function
```
So update the functions to look like this:
```Rust
#[debug_handler]
pub async fn new_user(
	Extension(pool): Extension<PgPool>,
	Json(user): Json<NewUser>
) -> Result<(StatusCode, Json<NewUser>), CustomError> {
	...
}
...
#[debug_handler]
pub async fn update_user(
	Path(id): Path<i32>,
	Extension(pool): Extension<PgPool>,
	Json(user): Json<UpdateUser>
) -> Result<(StatusCode, Json<UpdateUser>), CustomError> {
...
}
```

To make sure our app doesn't break immediately, let's seed the database with a couple of users. We'll add this function to our `util.rs` file:
```Rust
pub async fn seed_users(pool: PgPool) -> Result<(), CustomError> {
	let sql = "INSERT INTO users (id, first_name, last_name, email) VALUES ($1, $2, $3, $4) ON CONFLICT DO NOTHING";
	let user1 = User {
		id: 1,
		first_name: "John".to_string(),
		last_name: "Doe".to_string(),
		email: "john_doe@email.com".to_string(),
	};
	let user2 = User {
		id: 2,
		first_name: "Jane".to_string(),
		last_name: "Doe".to_string(),
		email: "jane_doe@email.com".to_string(),
};

	for user in vec![user1, user2] {
		sqlx::query(&sql)
			.bind(&user.id)
			.bind(&user.first_name)
			.bind(&user.last_name)
			.bind(&user.email)
			.execute(&pool)
			.await
			.map_err(|_| {
				CustomError::InternalServerError
			})?;
	}

Ok(())
}
```

Next, update the `main.rs` file to seed the database:
```Rust
let pool = PgPoolOptions::new()
	.max_connections(50)
	.connect(&database_url)
	.await
	.map_err(|_| {
		CustomError::DatabaseError
	})?;

sqlx::migrate!().run(&pool).await.map_err(|_| CustomError::MigrationError)?;

if let Ok(env_mode) = env::var("ENV_MODE") {
	if env_mode == "dev" {
		util::seed_users(pool.clone()).await?;
	}
}

let app = Router::new()
	.route("/", get(root))
	.route("/hello/:name", get(json_hello))
	.route("/users", get(model::user::all_users))
	.route("/user", post(model::user::new_user))
	.route("/user/:id", get(model::user::get_user))
	.route("/user/:id", put(model::user::update_user))
	.route("/user/:id", delete(model::user::delete_user))
	.layer(Extension(pool));
```

Of course, there's some new custom errors, so over to `errors.rs` to add them:
```Rust
#[derive(Debug)]
pub enum CustomError {
	BadRequest,
	UserNotFound,
	InternalServerError,
	DatabaseError,
	MigrationError,
}

impl IntoResponse for CustomError {
	fn into_response(self) -> axum::response::Response {
		let (status, error_message) = match self {
			Self::InternalServerError => (
				StatusCode::INTERNAL_SERVER_ERROR, "Internal Server Error",
			),
			Self::BadRequest=> (StatusCode::BAD_REQUEST, "Bad Request"),
			Self::UserNotFound => (StatusCode::NOT_FOUND, "User Not Found"),
			Self::DatabaseError => (StatusCode::INTERNAL_SERVER_ERROR, "Could not connect to database"),
			Self::MigrationError => (StatusCode::INTERNAL_SERVER_ERROR, "Could not migrate database"),
		};
		(status, Json(json!({"error": error_message}))).into_response()
	}
}
```

That looks like it _should_ be everything that we need for the backend, but I'm sure something will break. Let's go ahead and run `docker-compose up` to make sure everything is running smoothly.

Well, I was hoping to go a little bit longer before something broke, but that's ok. This shouldn't actually be breaking for you since I amended my `docker-compose.yml` file at the beginning. Basically, I had originally forgotten to attach a volume to the database container so that it would persist in-between sessions. 

Now that I've fixed that, let's head on over to the frontend directory to start wiring things up. We're just going to be adding a simple table that will list our users, so feel free to use whatever library you prefer for creating the table. I'll be using [AG Grid](https://www.ag-grid.com/vue-data-grid/getting-started/), a framework-agnostic table library that we've started using for some projects at work. With our docker containers still running, run the following commands to install the dependencies:
```bash
> docker exec -it rust-fullstack-frontend npm install ag-grid-community
> docker exec -it rust-fullstack-frontend npm install ag-grid-vue3
```

Create the table component in `rust-fullstack-frontend/src/components/DataTable.vue`:
```Vue
<script setup lang="ts">
import "ag-grid-community/styles/ag-grid.css";
import "ag-grid-community/styles/ag-theme-alpine.css";
import { AgGridVue } from "ag-grid-vue3";
import { ref } from 'vue'  

const users = ref([]);
const columns = [
	{
		field: "id",
	},
	{
		field: "first_name",
	},
	{
		field: "last_name",
	},
	{
		field: "email",
	},
];

fetch('http://localhost:3000/users')
	.then(response => response.json())
	.then(data => users.value = data);
</script>

<template>
	<ag-grid-vue
		v-if="users.length"
		class="ag-theme-alpine-dark"
		style="height: 200px"
		:rowData="users"
		:columnDefs="columns"
    >
	</ag-grid-vue>
</template>
```
Add a new view in `src/views/DataView.vue` with just our component:
```Vue
<script setup lang="ts">
import DataTable from '../components/DataTable.vue'
</script>

<template>
	<main>
		<DataTable />
	</main>
</template>
```
And finally add it to the router `src/router/index.ts`:
```TypeScript
import { createRouter, createWebHistory } from 'vue-router'
import HomeView from '../views/HomeView.vue'
import AboutView from '../views/AboutView.vue'
import DataView from '../views/DataView.vue'

const router = createRouter({
	history: createWebHistory(import.meta.env.BASE_URL),
	routes: [
		{
			path: '/',
			name: 'home',
			component: HomeView
		},
		{
			path: '/about',
			name: 'about',
			component: AboutView
		},
		{
			path: '/data',
			name: 'data',
			component: DataView
		}
	]
})

export default router
```
and the existing `RouterLink`s in `src/App.vue`:
```Vue
	...
	<RouterLink to="/data">Data</RouterLink>
	...
```

That should be everything, so we'll head over to the `/data` page to check out our new table. Which, of course, does not have data in it. Ok, so let's check the console. Ah, we don't have CORS enabled, so our backend is actually blocking the request. Back over to the backend to fix this!

While we're over here, we're actually going to add a couple of features: better logging and CORS. Better logging will let us monitor each of the requests as they come in, which will be super helpful if something breaks.

First, update our `Cargo.toml` file:
```TOML
[dependencies]
...
tower-http = { version = "0.4.4", features = ["trace", "cors"] }
```
Then, update `main.rs`:
```Rust
use tower_http::trace::TraceLayer;
use tower_http::cors::CorsLayer;
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

...

#[tokio::main]
async fn main() -> Result<(), CustomError> {
	let database_url = util::get_database_url();

	tracing_subscriber::registry()
		.with(tracing_subscriber::EnvFilter::new(
			std::env::var("tower_http=trace")
			.unwrap_or_else(|_| "example_tracing_aka_logging=debug,tower_http=debug".into()),
		))
		.with(tracing_subscriber::fmt::layer())
		.init();

...

	let app = Router::new()
		.route("/", get(root))
		.route("/hello/:name", get(json_hello))
		.route("/users", get(model::user::all_users))
		.route("/user", post(model::user::new_user))
		.route("/user/:id", get(model::user::get_user))
		.route("/user/:id", put(model::user::update_user))
		.route("/user/:id", delete(model::user::delete_user))
		.layer(Extension(pool))
		.layer(CorsLayer::permissive())
		.layer(TraceLayer::new_for_http());
...
}
```

And voila! Our app is now working!

## Conclusion
There's still a lot more that could be done with this app (like adding functionality to the frontend to create new users), but I think this is a solid foundation for what ideally would be a template for future projects. This was a great learning experience for me and hopefully for everyone that decides to read all of this. 

If you're interested in fleshing out the frontend more, I definitely recommend [Harness-Vue](https://github.com/RTIInternational/harness-vue) and [Harness-Vue-Bootstrap](https://github.com/RTIInternational/harness-vue-bootstrap). Harness-Vue is a dashboard state-management plugin for Vue and Harness-Vue-Bootstrap is a component library designed to be used in conjunction. This is a bit of a shameless plug since I helped work on both of them, but they're also what we use for almost all of our dashboards at work.

If your interests lie more in the backend, there's a ton of web frameworks that are built in Rust. I recommend both of these blog posts for more information:
- https://www.shuttle.rs/blog/2023/08/23/rust-web-framework-comparison
- https://blog.logrocket.com/top-rust-web-frameworks/

Follow the directions in `README.md` if you're interested in installing the finished project to try out. If you run into any bugs or if you have any questions, feel free to create an issue!
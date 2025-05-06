[<img alt="github" src="https://img.shields.io/badge/github-tfiala?style=for-the-badge&labelColor=555555&logo=github" height="20">](https://github.com/tfiala/tfiala-mongodb-migrator)
[<img alt="crates.io" src="https://img.shields.io/crates/v/tfiala-mongodb-migrator.svg?style=for-the-badge&color=fc8d62&logo=rust" height="20">](https://crates.io/crates/tfiala-mongodb-migrator)
[<img alt="docs.rs" src="https://img.shields.io/badge/docs.rs-66c2a5?style=for-the-badge&labelColor=555555&logoColor=white&logo=docs.rs" height="20">](https://docs.rs/tfiala-mongodb-migrator/latest/tfiala-mongodb_migrator)
[<img alt="build status" src="https://img.shields.io/github/actions/workflow/status/tfiala/tfiala-mongodb-migrator/rust.yml?branch=main&style=for-the-badge" height="20">](https://github.com/tfiala/tfiala-mongodb-migrator/actions/workflows/rust.yml)
[<img alt="codecov.io" src="https://img.shields.io/codecov/c/github/tfiala/tfiala-mongodb-migrator?style=for-the-badge" height="20">](https://codecov.io/gh/tfiala/tfiala-mongodb-migrator)

Mongodb migrations management tool.

## NOTE: this is a fork of [mongodb_migrator](https://github.com/kakoc/mongodb_migrator)

This is a (hopefully temporary) fork of the excellent work done by the
Konstantin Matsiushonak (<k.matushonok@gmail.com>) at [kakoc/mongodb_migrator](https://github.com/kakoc/mongodb_migrator).

The primary changes:

* updated all dependencies to latest, notably the mongodb driver (2.x -> 3.x)
* updated Rust version 2024

Expect this fork to go away once the dependency updates are integrated into the
authoritative repo.

## Setup

```toml
[dependencies]
tfiala-mongodb-migrator = "0.2.5"
```

## Functionality

* [Execute Rust based migrations][1]
* [Execute JavaScript based migrations][2]
* [Run as library][4]
* [Run as RESTful service][3]

[1]: https://github.com/kakoc/mongodb_migrator/blob/main/examples/as_lib.rs
[2]: https://github.com/kakoc/mongodb_migrator/blob/main/tests/shell/mod.rs
[3]: https://github.com/kakoc/mongodb_migrator/blob/main/tests/server/mod.rs
[4]: https://github.com/kakoc/mongodb_migrator/blob/main/tests/basic/mod.rs

## How to use

```rust
use anyhow::Result;
use async_trait::async_trait;
use mongodb::Database;
use serde_derive::{Deserialize, Serialize};
use testcontainers_modules::{
    mongo::Mongo,
    testcontainers::{runners::AsyncRunner, ContainerAsync},
};

use tfiala_mongodb_migrator::{
    migration::Migration,
    migrator::{
        default::DefaultMigrator,
        env::Env,
    },
};

#[tokio::main]
async fn main() -> Result<()> {
    let node = Mongo::default().start().await.unwrap();
    let host_port = node.get_host_port_ipv4(27017).await.unwrap();
    let url = format!("mongodb://localhost:{}/", host_port);
    let client = mongodb::Client::with_uri_str(url).await.unwrap();
    let db = client.database("test");

    let migrations: Vec<Box<dyn Migration>> = vec![Box::new(M0 {}), Box::new(M1 {})];
    DefaultMigrator::new()
        .with_conn(db.clone())
        .with_migrations_vec(migrations)
        .up()
        .await?;

    Ok(())
}

struct M0 {}
struct M1 {}

#[async_trait]
impl Migration for M0 {
    async fn up(&self, env: Env) -> Result<()> {
        let db = env.db().await?;
        db.collection("users")
            .insert_one(bson::doc! { "name": "Batman" })
            .await?;

        Ok(())
    }
}

#[async_trait]
impl Migration for M1 {
    async fn up(&self, env: Env) -> Result<()> {
        let db = env.db().await?;
        db.collection::<Users>("users")
            .update_one(
                bson::doc! { "name": "Batman" },
                bson::doc! { "$set": { "name": "Superman" } },
            )
            .await?;

        Ok(())
    }
}

#[derive(Serialize, Deserialize)]
struct Users {
    name: String,
}
```

## Roadmap

* [x] Rust based migrations
* [x] JavaScript based migrations
* [x] Logging
* [x] Rollbacks
* [ ] Cli tool
* [ ] UI dashboard
* [x] RESTful service
* [ ] As npm package
* [ ] Stragegies
  * [x] Fail first
  * [ ] Try all
  * [ ] Retries

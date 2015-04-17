# PostgreSQL for Schemamama

A PostgreSQL adapter for the lightweight database migration system
[Schemamama](https://github.com/SkylerLipthay/schemamama). Depends on the
`postgres` crate.

## Installation

If you're using Cargo, just add Schemamama to your `Cargo.toml`:

```toml
[dependencies]
schemamama = "*"
schemamama_postgres = "*"
postgres = "0.8"
```

## Usage

First, define some migrations:

```rust
#[macro_use]
extern crate schemamama;
extern crate schemamama_postgres;
extern crate postgres;

use schemamama::{Migration, Migrator};
use schemamama_postgres::{PostgresAdapter, PostgresMigration};

struct CreateUsers;
// Instead of using sequential numbers (1, 2, 3...), you may instead choose to use a global
// versioning scheme, such as epoch timestamps.
migration!(CreateUsers, 1, "create users table");

impl PostgresMigration for CreateUsers {
    fn up(&self, transaction: &postgres::Transaction) {
        transaction.execute("CREATE TABLE users (id BIGINT PRIMARY KEY);", &[]).unwrap();
    }

    fn down(&self, transaction: &postgres::Transaction) {
        transaction.execute("DROP TABLE users;", &[]).unwrap();
    }
}

struct CreateProducts;
migration!(CreateProducts, 2, "create products table");

impl PostgresMigration for CreateProducts {
    // ...
}
```

Then, run the migrations!

```rust
let url = "postgres://postgres@localhost";
let connection = postgres::Connection::connect(url, &SslMode::None).unwrap();
let adapter = PostgresAdapter::new(&connection);
// Create the metadata tables necessary for tracking migrations. This is safe to call more than
// once (`CREATE TABLE IF NOT EXISTS schemamama` is used internally):
adapter.setup_schema();

let mut migrator = Migrator::new(adapter);

migrator.register(Box::new(CreateUsers));
migrator.register(Box::new(CreateProducts));

// Execute migrations up to and including version 2:
migrator.up(2);
assert_eq!(migrator.current_version(), Some(2));

// Reverse all migrations:
migrator.down(None);
assert_eq!(migrator.current_version(), None);
```

## Testing

To run `cargo test`, you must have PostgreSQL running locally with a user role
named `postgres` with login access to a database named `postgres`. All tests
will work in the `pg_temp` schema, so the database will not be modified.

## To-do

* Make metadata table name configurable (currently locked in to `schemamama`).
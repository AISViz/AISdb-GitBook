---
hidden: true
---

# 🖥️ Detailed Start

This document explains how to install and use the AISdb web application components, providing instructions on running these services independently for testing or integration into an existing setup.

The application has three primary components:

> * Database server (Back end web API)
> * Database storage (Postgres database)
> * Web application interface (JS front end)

And some secondary components:

> * Documentation webserver
> * AIS receiver client
> * AIS livestream proxy dispatcher

### Dependencies

The following software is requisite for each AISDB service:

* Database Storage
  * Postgresql Database Server
  * Postgresql Database Client Libraries
  * See the [Postgres Install Tutorial](https://www.postgresql.org/docs/current/tutorial-install.html)
* Database Server
  * Rustup, the Rust Compiler Toolchain [Install Rust](https://www.rust-lang.org/tools/install)
  * OpenSSL
* Web Application Front End
  * Rustup, the Rust Compiler Toolchain [Install Rust](https://www.rust-lang.org/tools/install)
  * Binaryen, the WebAssembly Compiler Toolchain [Binaryen](https://github.com/WebAssembly/binaryen)
  * Wasm-pack, the Rust WebAssembly Packaging Utility [Install wasm-pack](https://rustwasm.github.io/wasm-pack/installer/)
  * Clang, the C/C++ Compiler [Clang Download](https://releases.llvm.org/download.html)
  * OpenSSL Development Toolkit (e.g. `libssl-dev` on ubuntu/debian)
  * Pkg-config [pkg-config](https://en.wikipedia.org/wiki/Pkg-config)
  * NodeJS, the JavaScript Runtime Environment [Node.js download](https://nodejs.org/en)
* Documentation Server
  * Python [Download Python](https://www.python.org/downloads/)
  * Rustup, the Rust Compiler Toolchain [Install Rust](https://www.rust-lang.org/tools/install)
  * Sphinx Doc [Installing and Running Sphinx](https://www.sphinx-doc.org/en/master/#get-started)
  * Maturin Build System [Maturin User Guide](https://www.maturin.rs/)
  * NodeJS, the JavaScript Runtime Environment [Node.js download](https://nodejs.org/en)
* AIS Receiver Client
  * Rustup, the Rust Compiler Toolchain [Install Rust](https://www.rust-lang.org/tools/install)
* AIS Proxy Dispatcher
  * Rustup, the Rust Compiler Toolchain [Install Rust](https://www.rust-lang.org/tools/install)

### Database Storage

Ensure that the Postgres server is running by following the [Postgres Database Server Tutorial](https://www.postgresql.org/docs/current/server-start.html). The other web services will use this server for storage and retrieval of AIS data.

### Database Server

Configure the database server by setting the following environment variables for the postgres database connection:

```
PGPASSFILE=$HOME/.pgpass
PGUSER="postgres"
PGHOST="[fc00::9]"
PGPORT="5432"
```

Navigate to the `database_server` folder in the project repository, install it with cargo, and then run it.

```
cd database_server
cargo install --path .
aisdb-db-server
```

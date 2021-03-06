# sqlx-adapter

[![Crates.io](https://img.shields.io/crates/v/sqlx-adapter.svg)](https://crates.io/crates/sqlx-adapter)
[![Docs](https://docs.rs/sqlx-adapter/badge.svg)](https://docs.rs/sqlx-adapter)
[![CI](https://github.com/casbin-rs/sqlx-adapter/workflows/CI/badge.svg)](https://github.com/casbin-rs/sqlx-adapter/actions)
[![codecov](https://codecov.io/gh/casbin-rs/sqlx-adapter/branch/master/graph/badge.svg)](https://codecov.io/gh/casbin-rs/sqlx-adapter)


Sqlx Adapter is the [Sqlx](https://github.com/launchbadge/sqlx) adapter for [Casbin-rs](https://github.com/casbin/casbin-rs). With this library, Casbin can load policy from Sqlx supported database or save policy to it with fully asynchronous support.

Based on [Sqlx](https://github.com/launchbadge/sqlx), The current supported databases are:

- [Mysql](https://www.mysql.com/)
- [Postgres](https://github.com/lib/pq)

## Install

Add it to `Cargo.toml`

```rust
sqlx-adapter = { version = "0.2.6", features = ["postgres"] }
async-std = "1.6.2"
```

## Configure

1. Set up database environment
   
    You must prepare the database environment so that `Sqlx` can do static check with queries during compile time. One convenient option is using docker to get your database environment ready:
    
    ```bash
    #!/bin/bash

    DIS=$(lsb_release -is)

    command -v docker > /dev/null 2>&1 || {
        echo "Please install docker before running this script." && exit 1;
    }

    if [ $DIS == "Ubuntu" ] || [ $DIS == "LinuxMint" ]; then
        sudo apt install -y \
            libpq-dev \
            libmysqlclient-dev \
            postgresql-client \
            mysql-client-core;

    elif [ $DIS == "Deepin" ]; then
        sudo apt install -y \
            libpq-dev \
            libmysql++-dev \
            mysql-client \
            postgresql-client;
    elif [ $DIS == "ArchLinux" ] || [ $DIS == "ManjaroLinux" ]; then
        sudo pacman -S libmysqlclient \
            postgresql-libs \
            mysql-clients \;
    else
        echo "Unsupported system: $DIS" && exit 1;
    fi

    docker run -itd \
        --restart always \
        -e POSTGRES_USER=casbin_rs \
        -e POSTGRES_PASSWORD=casbin_rs \
        -e POSTGRES_DB=casbin \
        -p 5432:5432 \
        -v /srv/docker/postgresql:/var/lib/postgresql \
        postgres:11;

    docker run -itd \
        --restart always \
        -e MYSQL_ALLOW_EMPTY_PASSWORD=yes \
        -e MYSQL_USER=casbin_rs \
        -e MYSQL_PASSWORD=casbin_rs \
        -e MYSQL_DATABASE=casbin \
        -p 3306:3306 \
        -v /srv/docker/mysql:/var/lib/mysql \
        mysql:8 \
        --default-authentication-plugin=mysql_native_password;

    ```

2. Create table `casbin_rules`

    ```bash
    # PostgreSQL
    psql postgres://casbin_rs:casbin_rs@127.0.0.1:5432/casbin -c "CREATE TABLE IF NOT EXISTS casbin_rules (
        id SERIAL PRIMARY KEY,
        ptype VARCHAR NOT NULL,
        v0 VARCHAR NOT NULL,
        v1 VARCHAR NOT NULL,
        v2 VARCHAR NOT NULL,
        v3 VARCHAR NOT NULL,
        v4 VARCHAR NOT NULL,
        v5 VARCHAR NOT NULL,
        CONSTRAINT unique_key_sqlx_adapter UNIQUE(ptype, v0, v1, v2, v3, v4, v5)
        );"

    # MySQL
    mysql -h 127.0.0.1 -u casbin_rs -pcasbin_rs casbin 

    CREATE TABLE IF NOT EXISTS casbin_rules (
        id INT NOT NULL AUTO_INCREMENT,
        ptype VARCHAR(12) NOT NULL,
        v0 VARCHAR(128) NOT NULL,
        v1 VARCHAR(128) NOT NULL,
        v2 VARCHAR(128) NOT NULL,
        v3 VARCHAR(128) NOT NULL,
        v4 VARCHAR(128) NOT NULL,
        v5 VARCHAR(128) NOT NULL,
        PRIMARY KEY(id),
        CONSTRAINT unique_key_sqlx_adapter UNIQUE(ptype, v0, v1, v2, v3, v4, v5)
        ) ENGINE=InnoDB DEFAULT CHARSET=utf8;
    ```

3. Configure `env`

    Rename `sample.env` to `.env` and put `DATABASE_URL`, `POOL_SIZE`   inside

    ```bash
    DATABASE_URL=postgres://casbin_rs:casbin_rs@localhost:5432/casbin
    # DATABASE_URL=mysql://casbin_rs:casbin_rs@localhost:3306/casbin
    POOL_SIZE=8
    ```

    Or you can export `DATABASE_URL`, `POOL_SIZE` (When you are building project, you must use export env way)

    ```bash
    export DATABASE_URL=postgres://casbin_rs:casbin_rs@localhost:5432/casbin
    export POOL_SIZE=8
    ```


## Example

```rust
use sqlx_adapter::casbin::prelude::*;
use sqlx_adapter::casbin::Result;
use sqlx_adapter::SqlxAdapter;

#[async_std::main]
async fn main() -> Result<()> {
    let m = DefaultModel::from_file("examples/rbac_model.conf").await?;
    
    let a = SqlxAdapter::new().await?;
    let mut e = Enforcer::new(m, a).await?;
    
    Ok(())
}

```

## Features

- `postgres`
- `mysql`

*Attention*: `postgres` and `mysql` are mutual exclusive which means that you can only activate one of them. Currently we don't have support for `sqlite`, it may be added in the near future.

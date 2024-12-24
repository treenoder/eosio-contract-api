# EOSIO Contract API
check ci
The aim of this project is to provide a framework to fill and query state and history for specific
contracts on eosio based blockchains. 

This project uses the eosio State History Plugin as data source and PostgreSQL to store / query the data.
Per block transactions guarantee that the database is consistent at any time.

## Requirements

* NodeJS >= 16.0
* PostgreSQL >= 13.0
  * You need to enable the `pg_trgm` extension with `CREATE EXTENSION pg_trgm;`
* Redis >= 5.0
* Nodeos >= 1.8.0 (only tested with 2.0 and 2.1) The state history plugin needs to be enabled and the options: 
`trace-history = true`, `chain-state-history = true`

Suggestions
* Hasura GraphQL Engine >= 1.3 (if you want to allow GraphQL queries)
* PGAdmin 4 (Interface to manage the postgres database)

## Configuration
The config folder contains 3 different configuration files

#### connections.config.json
This file contains Postgres / Redis / Nodeos connection data for the used chain.

Notes
* Redis: Can be used for multiple chains without further action
* PostgreSQL: Each chain needs it own postgres database (can use the same postgres instance), but multiple readers of the same
chain can use the same database if they are non conflicting
* Nodeos: nodeos should habe a full state history for the range you are trying to index

```json
{
  "postgres": {
    "host": "127.0.0.1",
    "port": 5432,
    "user": "username",
    "password": "changeme",
    "database": "api-wax-mainnet-atomic-1"
  },
  "redis": {
    "host": "127.0.0.1",
    "port": 6379
  },
  "chain": {
    "name": "wax-mainnet",
    "chain_id": "1064487b3cd1a897ce03ae5b6a865651747e2e152090f99c1d19d44e01aea5a4",
    "http": "http://127.0.0.1:8888",
    "ship": "ws://127.0.0.1:8080"
  }
}
```

#### readers.config.json
This file is used to configure the filler

For atomicassets / atomicmarket you should specify the following start blocks
* `wax-mainnet`: `64000000`
* `wax-testnet`: `35795440` (Here you need to use it otherwise it will break)
* `eos-mainnet`: `99070000`
* `proton-mainnet`: `50289000`
* `proton-testnet`: `53440000`

```json5
[
  // Multiple Readers can be defined and each one will run in a separated thread
  {
    "name": "atomic-1", // Name of the reader. Should be unique per chain and should not change after it was started

    "start_block": 0, // start at a specific block. If ready was already started, this can only be higher than the last indexed block
    "stop_block": 0, // stop at a specific block
    "irreversible_only": false, // If you need data for a lot of contracts and do not need live data, this option is faster

    "ship_prefetch_blocks": 50, // How many unconfirmed blocks ship will send
    "ship_min_block_confirmation": 30, // After how many blocks the reader will confirm the blocks
    "ship_ds_queue_size": 20, // how many blocks the reader should pre-serialize the action / table data
      
    "ds_ship_threads": 4, // How many threads should be used to deserialize traces and table deltas

    "db_group_blocks": 10, // In catchup mode, the reader will group this amount of bl

    "contracts": [
      // AtomicAssets handler which provides data for the AtomicAssets NFT standard
      {
        "handler": "atomicassets",
        "args": {
          "atomicassets_account": "atomicassets", // Account where the contract is deployed
          "store_logs": true, // store logs
          "store_transfers": true // store the transfer history
        }
      }
    ],
    
    "list_polls": [ // optional
      {
        url: 'https://example.com/lists', // endpoint for the lists
        api_key: '123', // send as X-API-Key header to the server
        frequency: 600, // optional, poll frequency in seconds, defaults to 10 minutes
      }
    ]
  }
]
```

#### server.config.json

```json5
{
  "provider_name": "pink.network", // Provider which is show in the endpoint documentation
  "provider_url": "https://pink.network",

  "server_addr": "0.0.0.0", // Server address to bind to
  "server_name": "wax.api.atomicassets.io", // Server name which is shown in the documentation
  "server_port": 9000, // Server Port

  "cache_life": 2, // GET endpoints are cached for this amount of time (in seconds)
  "trust_proxy": true, // Enable if you use a reverse proxy to have correct rate limiting by ip

  "rate_limit": {
    "interval": 60, // Interval to reset the counter (in seconds)
    "requests": 240 // How much requests can be made in the defined interval
  },
    
  "ip_whitelist": [], // These IPs are not rate limited or receive cached requests
  "slow_query_threshold": 7500, // If specific queries take longer than this threshold a warning is created

  "max_query_time_ms": 10000, // max execution time for a database query
  "max_db_connections": 50, // max number of concurrent db connections / db queries
        
  "namespaces": [
    // atomicassets namespace which provides an API for basic functionalities
    {
      "name": "atomicassets", 
      "path": "/atomicassets", // Each API endpoint will start with this path
      "args": {
        "atomicassets_account": "atomicassets" // Account where the contract is deployed
      }
    }
  ]
}

```

## Installation

This project consists of two separated processes which need to be started and stopped independently:
* The API which will provide the socket and REST endpoints (or whatever is used)
* The filler which will read the data from the blockchain and fills the database

The filler needs to be started before the API when running it for the first time:

Prerequisites:
- PostgreSQL
  - Create a database and user which is allowed to read and write on that db
    
- EOSIO node 
  - State History Plugin enabled with options `trace-history = true`, `chain-state-history = true`
  - Fully synced for the block range you want to process
  - Open socket and http api

- Copy and modify example configs with the correct connection params

There are two suggested ways to run the project: Docker if you want to containerize the application or PM2 if you want to run it on system level

### Docker

1. `git clone && cd eosio-contract-api`
2. There is an example docker compose file provided
3. `docker-compose up -d`

Start
* `docker-compose start eosio-contract-api-filler`
* `docker-compose start eosio-contract-api-server`

Stop
* `docker-compose stop eosio-contract-api-filler`
* `docker-compose stop eosio-contract-api-server`

### PM2

1. `git clone && cd eosio-contract-api`
2. `yarn install`
3. `yarn global add pm2`

Start
* `pm2 start ecosystems.config.json --only eosio-contract-api-filler`
* `pm2 start ecosystems.config.json --only eosio-contract-api-server`

Stop
* `pm2 stop eosio-contract-api-filler`
* `pm2 stop eosio-contract-api-server`

## Currently Supported Contracts

### Readers (used to fill the database)

Readers are used to fill the database for a specific contract.

#### atomicassets

```json5
{
  "handler": "atomicassets",
  "args": {
    "atomicassets_account": "atomicassets", // account where the atomicassets contract is deployed
    "store_transfers": true, // store the transfer history  
    "store_logs": true // store data structure logs
  }
}
```

#### atomicmarket
This reader requires a atomicassets and a delphioracle reader with the same contract as specified here

```json5
{
  "handler": "atomicmarket",
  "args": {
    "atomicassets_account": "atomicassets", // account where the atomicassets contract is deployed
    "atomicmarket_account": "atomicmarket", // account where the atomicmarket contract is deployed
    "store_logs": true // Store logs of sales / auctions
  }
}
```

#### delphioracle

```json5
{
  "handler": "delphioracle",
  "args": {
    "delphioracle_account": "delphioracle" // account where the delphioracle contract is deployed
  }
}
```

### Namespace (API endpoints)

A namespace provides an API for a specific contract or use case and is based on data a reader provides

#### atomicassets

```json5
{
  "handler": "atomicassets",
  "args": {
    "atomicassets_account": "atomicassets", // account where the atomicassets contract is deployed
    "connected_reader": "atomic-1" // reader to which the API connects for live data
  }
}
```

#### atomicmarket

```json5
{
  "handler": "atomicmarket",
  "args": {
    "atomicmarket_account": "atomicmarket", // account where the atomicmarket contract is deployed
    "connected_reader": "atomic-1" // reader to which the API connects for live data
  }
}
```
## Testing

To run the test on this project:

1. In the `config` folder, copy the `example-*` files and rename them to `connections.config.json`
`readers.config.json` and `server.config.json`
2. Start the redis and postgresql servers, you can do that by using 
docker compose (`docker compose up eosio-contract-api-redis eosio-contract-api-postgres`)
or installing them directly into your computer.
3. Modify the `connection.config.json` to point to your local computer and have the correct credentials.
4. Execute the init-test-db script using the command `yarn dev:init-test-db`
5. Run the test using the command `yarn dev:test`

### Developing new features

1. Make sure that everything in our code base is working properly, ej: Test running and application compiling.
2. Depending on your way of working, develop the feature or the bug fix.
3. Add some test cases to cover up your code.
4. Test it manually in using the API.
5. Create a PR and fix the comments of the reviewer.
6. Merge and deploy.


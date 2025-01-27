# Rediseen

[!["Latest Release"](https://img.shields.io/github/release/xd-deng/rediseen.svg)](https://github.com/xd-deng/rediseen/releases/latest)
[![action](https://github.com/xd-deng/rediseen/workflows/Rediseen/badge.svg)](https://github.com/XD-DENG/rediseen/actions)
[![travis](https://api.travis-ci.org/XD-DENG/rediseen.svg?branch=master)](https://travis-ci.org/XD-DENG/rediseen/branches)
[![codecov](https://codecov.io/gh/XD-DENG/rediseen/branch/master/graph/badge.svg)](https://codecov.io/gh/XD-DENG/rediseen)
[![License](https://img.shields.io/:license-apache2-green.svg)](http://www.apache.org/licenses/LICENSE-2.0.html)
[![Go Report Card](https://goreportcard.com/badge/github.com/xd-deng/rediseen)](https://goreportcard.com/report/github.com/xd-deng/rediseen)


Start a REST-like API service for your Redis database, without writing a single line of code.

- Allows clients to query records in Redis database via HTTP conveniently
- Allows you to specify which logical DB(s) to expose, and what key patterns to expose
- Expose results of [Redis `INFO` command](https://redis.io/commands/info) in nice format, so **you can use `Rediseen` as a connector between your Redis DB and monitoring dashboard** as well.
- Supports API Key authentication

(Inspired by [sandman2](https://github.com/jeffknupp/sandman2), and built on shoulder of [go-redis/redis
](https://github.com/go-redis/redis))

<p align="center"> 
    <a href="https://youtu.be/SpHNnPIT0HM">
        <img src="https://raw.githubusercontent.com/XD-DENG/rediseen/misc/images/rediseen_video_demo.png" alt="drawing" width="450"/>
    </a>
</p>

- [1. Quick Start](#1-quick-start)
  - [1.1 Quick Start with Homebrew](#11-quick-start-with-homebrew)
  - [1.2 Quick Start with Docker](#12-quick-start-with-docker)
- [2. Usage](#2-usage)
  - [2.1 How to Install](#21-how-to-install)
  - [2.2 How to Configure](#22-how-to-configure)
  - [2.3 How to Start the Service](#23-how-to-start-the-service)
  - [2.4 How to Consume the Service](#24-how-to-consume-the-service)
- [3. Authentication](#3-authentication)
- [4. License](#4-license)
- [5. Reference](#5-reference)


## 1. Quick Start

Let's assume that your Redis database URI is `redis://:@localhost:6379`, and you want to expose keys with prefix `key:` in logical database `0`.

### 1.1 Quick Start with Homebrew

```bash
# Install using Homebrew
brew install XD-DENG/rediseen/rediseen

# Configuration
export REDISEEN_REDIS_URI="redis://:@localhost:6379"
export REDISEEN_DB_EXPOSED=0
export REDISEEN_KEY_PATTERN_EXPOSED="^key:([0-9a-z]+)"

# Start the service
rediseen start
```

Now you should be able to query against your Redis database, like `http://localhost:8000/0`, `http://localhost:8000/0/key:1`,
`http://localhost:8000/info` or `http://localhost:8000/info/server`
(say you have keys `key:1` (string) and `key:2` (hash) set in your logical DB `0`). Sample responses follow below

```bash
GET /0

{
    "count": 2,
    "total": 2,
    "keys": [
        {
            "key": "key:1",
            "type": "string"
        },
        {
            "key": "key:2",
            "type": "hash"
        }
    ]
}
```

```bash
GET /0/key:1

{
    "type": "string",
    "value": "rediseen"
}
```

```bash
GET /info

{
    Server: {
        redis_version: "5.0.6",
        ...
    },
    Clients: {
        ...
    },
    Replication: {
        ...
    },
    ...
}
```

```bash
GET /info/server

{
    Server: {
        redis_version: "5.0.6",
        ...
    }
}
```

For more details, please refer to the rest of this README documentation.

### 1.2 Quick Start with Docker

```bash
docker run \
    -e REDISEEN_REDIS_URI="redis://:@redis_host:6379" \
    -e REDISEEN_DB_EXPOSED=0 \
    -e REDISEEN_KEY_PATTERN_EXPOSED="^key:([0-9a-z]+)" \
    -p 8000:8000 \
    xddeng/rediseen:latest
```

Please note:
- If you want `Rediseen` to be accessible only from the Docker host, use `-p 127.0.0.1:8000:8000` instead of `-p 8000:8000`
    (configuration item `REDISEEN_HOST` would not help when you run `Rediseen` with Docker). 
- `REDISEEN_REDIS_URI` above should contain a specific host address. If you are running Redis database using Docker
    too, you can consider using Docker's `link` or `network` feature to ensure connectivity between Rediseen and your Redis database (refer to the
    example below).
- You can choose the image tag among `latest` (latest release version), `nightly` (latest code in master branch), `unstable` (latest dev branch),
    and release tags (like `2.1.0`. Check [Docker Hub/xddeng/rediseen](https://hub.docker.com/r/xddeng/rediseen/tags)
    for full tag list)
    
A full example using Docker follows below

```bash
docker network create test-net

docker run -d --network=test-net --name=redis-server redis

docker run \
    -d --network=test-net \
    -e REDISEEN_REDIS_URI="redis://:@redis-server:6379" \
    -e REDISEEN_DB_EXPOSED=0 \
    -e REDISEEN_KEY_PATTERN_EXPOSED="^key:([0-9a-z]+)" \
    -p 8000:8000 \
    xddeng/rediseen:latest

curl -s http://localhost:8000/0
```

Result is like 

```
{
  "count": 0,
  "total": 0,
  "keys": null
}
```

Then you can execute

```bash
docker exec -i redis-server redis-cli set key:0 100

curl -s http://localhost:8000/0
```

and you can expect output below

```
{
  "count": 1,
  "total": 1,
  "keys": [
    {
      "key": "key:0",
      "type": "string"
    }
  ]
}
```


## 2. Usage


### 2.1 How to Install 

You can choose to install `Rediseen` either using `Homebrew` or from source.

- **Install Using `Homebrew`**

You can use [Homebrew](https://brew.sh/) to install `Rediseen`, no matte you are using `macOS`, or `Linux`/
`Windows 10 Subsystem for Linux` ([how to install Homebrew](https://docs.brew.sh/Installation)).

```bash
brew install XD-DENG/rediseen/rediseen

rediseen help
```

- **Build from source** (with Go 1.12 or above installed)

You can also build `Rediseen` from source.

```bash
git clone https://github.com/XD-DENG/rediseen.git
cd rediseen
go build . # executable binary file "rediseen" will be created

./rediseen help
```


### 2.2 How to Configure

Configuration is done via **environment variables**.

| Item | Description | Remark |
| --- | --- | --- |
| `REDISEEN_REDIS_URI` | URI of your Redis database, e.g. `redis://:@localhost:6379` | Compulsory |
| `REDISEEN_HOST` | Host of the service. Host will be `localhost` if `REDISEEN_HOST` is not explicitly set. Set to `0.0.0.0` if you want to expose your service to the world. | Optional |
| `REDISEEN_PORT` | Port of the service. Default port is 8000. | Optional |
| `REDISEEN_DB_EXPOSED` | Redis logical database(s) to expose.<br><br>E.g., `0`, `0;3;9`, `0-9;15`, or `*` (expose all logical databases) | Compulsory |
| `REDISEEN_KEY_PATTERN_EXPOSED` | Regular expression pattern, representing the name pattern of keys that you intend to expose.<br><br>For example, `user:([0-9a-z/.]+)\|^info:([0-9a-z/.]+)` exposes keys like `user:1`, `user:x1`, `testuser:1`, `info:1`, etc. |  |
| `REDISEEN_KEY_PATTERN_EXPOSE_ALL` | If you intend to expose ***all*** your keys, set `REDISEEN_KEY_PATTERN_EXPOSE_ALL` to `true`. | `REDISEEN_KEY_PATTERN_EXPOSED` can only be empty (or not set) if you have set `REDISEEN_KEY_PATTERN_EXPOSE_ALL` to `true`. |
| `REDISEEN_API_KEY` | API Key for authentication. Authentication is only enabled when `REDISEEN_API_KEY` is set and is not "".<br><br>Once it is set, client must add the API key into HTTP header as field `X-API-KEY` in order to access the API.<br><br>Note this authentication is only considered secure if used together with other security mechanisms such as HTTPS/SSL [1]. | Optional |
| `REDISEEN_TEST_MODE` | Set to `true` to skip Redis connection validation for unit tests. | For Dev Only |


### 2.3 How to Start the Service

Run command below,

```bash
rediseen start
```

Then you can access the service at
- `http://<your server address>:<REDISEEN_PORT>/<redis DB>`
- `http://<your server address>:<REDISEEN_PORT>/<redis DB>/<key>`
- `http://<your server address>:<REDISEEN_PORT>/<redis DB>/<key>/<index or value or member>`

If you would like to run the service in daemon mode, apply flag `-d`.

```bash
# run service in daemon mode
rediseen -d start

# stop service running in daemon mode
rediseen stop
```


### 2.4 How to Consume the Service

#### 2.4.1 `/<redis DB>`

This endpoint will return response in which you can get
- the number of keys which are exposed
- keys exposed and their types (**only up to 1000 keys will be showed**)

A sample response follows below

```
{
    "count": 3,
    "total": 3,
    "keys": [
        {
            "key": "key:1",
            "type": "string"
        },
        {
            "key": "key:5",
            "type": "hash"
        },
        {
            "key": "key:100",
            "type": "zset"
        }
    ]
}
```

#### 2.4.2 `/<redis DB>/<key>`

| Data Type | Underlying Redis Command |
| --- | --- |
| STRING | `GET(key)` |
| LIST   | `LRANGE(key, 0, -1)` |
| SET    | `SMEMBERS(key)` |
| HASH   | `HGETALL(key)` |
| ZSET   | `ZRANGE(key, 0, -1)` |


#### 2.4.3 `/<redis DB>/<key>/<index or value or member>`

| Data Type | Usage | Return Value |
| --- | --- | --- |
| STRING | `/<redis DB>/<key>/<index>`  | `<index>`-th character in the string |
| LIST   | `/<redis DB>/<key>/<index>` | `<index>`-th element in the list |
| SET    | `/<redis DB>/<key>/<member>` | if `<member>` is member of the set |
| HASH   | `/<redis DB>/<key>/<field>` | value of hash `<field>` in the hash |
| ZSET   | `/<redis DB>/<key>/<memeber>` | index of `<member>` in the sorted set |

#### 2.4.4 `/info`

It returns ALL results from [Redis `INFO` command](https://redis.io/commands/info) as a nicely-formatted JSON object.

#### 2.4.5 `/info/<info_section>`

It returns results from [Redis `INFO <SECTION>` command](https://redis.io/commands/info) as a nicely-formatted JSON object.

Supported `info_section` values can be checked by querying `/info`. They vary according to your Redis version.


## 3. Authentication

API Key authentication is supported.

To enable authentication, simply set environment variable `REDISEEN_API_KEY` and the value would be the key.
Once it's set, client will have to add the API key as `X-API-KEY` in their HTTP header in order to access anything
meaningful, otherwise 401 error (`Unauthorized`) will be returned.

For example,

```bash
export REDISEEN_REDIS_URI="redis://:@localhost:6379"
export REDISEEN_DB_EXPOSED=0
export REDISEEN_KEY_PATTERN_EXPOSED="^key:([0-9a-z]+)"
export REDISEEN_API_KEY="demo_key" # Set REDISEEN_API_KEY to enforce API Key Authentication

# Start the service and run in background
rediseen -d start

# REJECTED: No X-API-KEY is given in HTTP header
curl -s http://localhost:8000/0 | jq
{
  "error": "unauthorized"
}

# REJECTED: Wrong X-API-KEY is given in HTTP header
curl -s -H "X-API-KEY: wrong_key" http://localhost:8000/0 | jq
{
  "error": "unauthorized"
}

# ACCEPTED: Correct X-API-KEY is given in HTTP header
curl -s -H "X-API-KEY: demo_key" http://localhost:8000/0 | jq
{
  "count": 1,
  "total": 1,
  "keys": [
    {
      "key": "key:1",
      "type": "rediseen"
    }
  ]
}
```


## 4. License

[Apache-2.0](https://www.apache.org/licenses/LICENSE-2.0)


## 5. Reference

[1] https://swagger.io/docs/specification/authentication/api-keys/

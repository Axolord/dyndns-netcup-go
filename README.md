# DYNDNS NETCUP GO
[![Build](https://github.com/Axolord/dyndns-netcup-go/actions/workflows/build.yml/badge.svg)](https://github.com/Axolord/dyndns-netcup-go/actions/workflows/build.yml)
[![Issues](https://img.shields.io/github/issues/Axolord/dyndns-netcup-go)](https://github.com/Axolord/dyndns-netcup-go/issues)
[![Release](https://img.shields.io/github/release/Axolord/dyndns-netcup-go?include_prereleases)](https://github.com/Axolord/dyndns-netcup-go/releases)
[![Go Report Card](https://goreportcard.com/badge/github.com/Axolord/dyndns-netcup-go)](https://goreportcard.com/report/github.com/Axolord/dyndns-netcup-go)

Dyndns client for the netcup DNS API written in go.
Forked and improved from [Hentra/dyndns-netcup-go](https://github.com/Hentra/dyndns-netcup-go).
Not related to netcup GmbH. 

## Table of Contents
<!-- vim-markdown-toc GFM -->

* [Features](#features)
* [Installation](#installation)
  * [Prequisites](#prequisites)
	* [Docker compose](#docker-compose-example-using-secret-files)
	* [Docker CLI](#docker-cli)
  * [Environment Variables](#environment-variables)
	* [Manually without Docker](#manually-from-source-without-docker)
* [Contributing](#contributing)

<!-- vim-markdown-toc -->

## Features

* Multi domain support
* Subdomain support
* TTL update support
* Creation of a DNS record if it doesn't already exist
* Multi host support (nice when you need to update both `@` and `*`) 
* IPv6 support

### Differnces from Hentra/dyndns-netcup-go
* improved Docker security (inspired by [wollomatic/socket-proxy](https://github.com/wollomatic/socket-proxy))
  * usage of Scratch base image instead of alpine
  * image runs as unpriviledged user with UID 62534 instead as root
  * support for [Docker Compose Secret Files](https://docs.docker.com/compose/how-tos/use-secrets/) 
* updated Go version and dependencies

If you need additional features please open up an
[Issue](https://github.com/Axolord/dyndns-netcup-go/issues).

## Installation 

### Prequisites

* You need to have a netcup account and a domain, obviously.
* Then you need an apikey and apipassword.
  [Here](https://www.netcup-wiki.de/wiki/CCP_API#Authentifizierung) is a
description (in German) on how you get those.

### Docker compose example using secret files

You need to create config.yml file in the same directory as the docker-compose.yml file, take a look at [config/example.yml](config/example.yml) for an example.

> [!WARNING]
> For a Docker setup do not save your secrets (api key, etc.) directly in the config.yml, but rather as environment variables or [docker compose secret files](https://docs.docker.com/compose/how-tos/use-secrets/), as shown below!

For secrets management create three files under `secrets/` with the names `customernr`, `apikey` and `apipassword`, like in the [secrets/](secrets/) directory.

> [!TIP]
> To further protect the secrets from unauthorized access, make sure it is owned by the user that runs dyndns-netcup-go, being the UID 62534 (see the [Dockerfile](build/package/Dockerfile)] and make it read-only:

```shell
sudo chown 62534:62534 secrets/*
sudo chmod 440 secrets/*
```

After that setup you can use the following docker-compose.yml as an example, also available in [docker-compose.yml](docker-compose.yml):
```compose.yml
services:
  dyn-dns:
    image: ghcr.io/hentra/dyndns-netcup-go:1
    container_name: Netcup-Dyn-DNS
    environment:
      - INTERVAL=300
      - CUSTOMERNR_FILE=/run/secrets/customernr
      - APIKEY_FILE=/run/secrets/apikey
      - APIPASSWORD_FILE=/run/secrets/apipassword
    secrets:
      - customernr
      - apikey
      - apipassword
    volumes:
      - ./config.yml:/config.yml
    security_opt:
      - no-new-privileges
    cap_drop:
      - ALL
    restart: unless-stopped
    networks:
      - ipv6-enabled-network

secrets:
  customernr:
    file: ./secrets/customernr
  apikey:
    file: ./secrets/apikey
  apipassword:
    file: ./secrets/apipassword

networks:
  ipv6-enabled-network:
    enable_ipv6: true
```

### Docker CLI

    docker run -d \
        -v $(pwd)/config.yml:/config.yml \
        -e INTERVAL=300 \
        -e CUSTOMERNR=111111 \
        -e APIKEY=my-fancy-api-key \
        -e APIPASSWORD=my-fancy-api-pw \
        ghcr.io/axolord/dyndns-netcup-go:1

This allows you to store the configuration in plain text(e.g. git) and inject the secrets safely from a secret management solution.

### Environment Variables

| Environment Variable | Description                                                             |
|----------------------|-------------------------------------------------------------------------|
| INTERVAL             | defines the interval of DNS updates in seconds                          |
| CUSTOMERNR_FILE      | location of the secrets file containing your customer number from netcup|
| APIKEY_FILE          | location of the secrets file containing your API key from netcup        |
| APIPASSWORD_FILE     | location of the secrets file containing your API password from netcu    |
| CUSTOMERNR           | alternative to secrets file: customer number from netcup                |
| APIKEY               | alternative to secrets file: containing your API key from netcup        |
| APIPASSWORD          | alternative to secrets file: containing your API password from netcu    |

### Manually from source without Docker

If you prefer to run the executeable directly without Docker or Podman, you have to build
the executeable manually.

> [!IMPORTANT]
> I am only partially supporting this setup, since it is inherited from upstream.
> If someone wants to help testing/maintaining this way of installation I am happy to reseive pull requests!
> But since I am not using it myself, I am not feeling good about providing only partially tested executeables.

With that out of the way, here are the steps:

First, install [Go](https://golang.org/doc/install) as
recommended.  After that run following commands:

    git clone https://github.com/Axolord/dyndns-netcup-go.git 
    cd dyndns-netcup-go
    go install

This will create a binary named `dyndns-netcup-go` and install it to your go
binary home. Make sure your `GOPATH` environment variable is set. 

#### Usage

Then, take the [example configuration](./config/example.yml) `config/example.yml` 
to `config.yml` and fill out all the fields. There are some comments in the file for further information. 
2. Run `dyndns-netcup-go -v` in the **same** directory as your configuration file and it will
configure your DNS Records. You can specify the location of the
configuration file with the `-c` or `-config` flag if you don't want to run
it in the same directory. To disable the output for information remove the `-v` flag. You will
still get the output from errors.

It might be necessary to run this program every few minutes. That interval
depends on how you configured your TTL.

#### Commandline flags

For a list of all available command line flags run `dyndns-netcup-go -h`.

## Contributing 

For any feature requests and or bugs open up an
[Issue](https://github.com/Axolord/dyndns-netcup-go/issues).  Feel free to also
add a pull request and I will have a look on it.

## Further Reading

### Docker and IPv6

Many people have problems understanding and using IPv6 in docker networks.
Using the provided example ipv6 support is enabled using NAT66 provided by Docker since version 27.
While NAT66 is not the optimal way to support IPv6, in a context where this DynDNS client is used, it is the preferred way in my opinion.
For further reading take a look at the blog post [The IPv6 situation on Docker is good now!](https://ounapuu.ee/posts/2024/12/20/docker-ipv6/).

### Caching

Without the cache the application would lookup its ip addresses and fetch the DNS records from netcup.
After that it will compare the specified hosts in the DNS records with the current ip addresses and update if necessary.
As reported in [this issue](https://github.com/Hentra/dyndns-netcup-go/issues/1) from upstream it would be also possible to store the ip addresses between two runs of the application and only fetch DNS records from netcup when they differ.

### Secrets and Docker Security

This fork got created because of the way secrets are handled by existing dyndns programs for netcup.
The upstream project is waiting for approval of a [PR to support secrets as environment variables](https://github.com/Hentra/dyndns-netcup-go/pull/18) since spring 2024,
[b2un0/docker-netcup-dyndns](https://github.com/b2un0/docker-netcup-dyndns) has support for environment variables, but no secret files
and its own upstream without docker support [stecklars/dynamic-dns-netcup-api](https://github.com/stecklars/dynamic-dns-netcup-api) also handles secrets only via a simple config file.

Furthermore, the mentioned projects, as well as many other alternatives, usually do not follow [docker security best practices](https://cheatsheetseries.owasp.org/cheatsheets/Docker_Security_Cheat_Sheet.html).
While not being that important for a dyndns client that only makes requests and does not permanently open any ports, I still want my containers
to have as little priviledges as possible.


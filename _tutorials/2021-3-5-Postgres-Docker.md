---
layout: post
title: "Running PostgreSQL Locally with Docker"
---

This post is a super quick-and-dirty look at how to run a local PostgreSQL instance with Docker. For some reason the documentation provided on the page for [the image](https://hub.docker.com/_/postgres?tab=description&page=1&ordering=last_updated) never works for me, so this is primarily to remind myself how I finally got the process to work.

### Walkthrough

Pull the Postgres Docker image from dockerhub. There are plenty of different tags to choose from, but here we'll be using the `postgres:alpine` tag, which is the latest release built on an Alpine Linux base image. 

```bash
$ docker pull postgres:alpine
```

Start the container:

```bash
$ docker run --rm --name postgres -e POSTGRES_PASSWORD=password -d postgres:alpine
```

When the container starts, it automatically starts the DBMS instance, and the default user and database are created. 

Now open an interactive shell within the container, and connect to the DBMS with `psql`:

```bash
# On the host machine
$ docker exec -it postgres bash

# Within the container
bash-5.1# psql -U postgres

# Now in the DBMS shell
postgres=# CREATE TABLE ...
```

Done. Happy hacking.

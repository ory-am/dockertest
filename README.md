# [ory.am](https://ory.am)/dockertest

[![Build Status](https://travis-ci.org/ory-am/dockertest.svg)](https://travis-ci.org/ory-am/dockertest?branch=master)
[![Coverage Status](https://coveralls.io/repos/ory-am/dockertest/badge.svg?branch=master&service=github)](https://coveralls.io/github/ory-am/dockertest?branch=master)

Use Docker to run your Go language integration tests against third party services on **Microsoft Windows, Mac OSX and Linux**!

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**

- [Why should I use Dockertest?](#why-should-i-use-dockertest)
- [Installing and using Dockertest](#installing-and-using-dockertest)
  - [Using Dockertest](#using-dockertest)
  - [Setting up Travis-CI](#setting-up-travis-ci)
- [Troubleshoot & FAQ](#troubleshoot-&-faq)
  - [Out of disk space](#out-of-disk-space)
  - [Removing old containers](#removing-old-containers)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Why should I use Dockertest?

When developing applications, it is often necessary to use services that talk to a database system.
Unit Testing these services can be cumbersome because mocking database/DBAL is strenuous. Making slight changes to the
schema implies rewriting at least some, if not all of the mocks. The same goes for API changes in the DBAL.
To avoid this, it is smarter to test these specific services against a real database that is destroyed after testing.
Docker is the perfect system for running unit tests as you can spin up containers in a few seconds and kill them when
the test completes. The Dockertest library provides easy to use commands for spinning up Docker containers and using
them for your tests.

## Installing and using Dockertest

Using Dockertest is straightforward and simple. Check the [releases tab](https://github.com/ory-am/dockertest/releases)
for available releases.

To install dockertest, run

```
go get gopkg.in/ory-am/dockertest.v3
```

### Using Dockertest


```go
package dockertest_test

import (
	"testing"
	"log"
	"gopkg.in/ory-am/dockertest.v3"
	_ "github.com/go-sql-driver/mysql"
	"database/sql"
	"fmt"
	"os"
)

var dockerEndpoint = "http://localhost:2375" // this is the default
var db *sql.DB

func TestMain(m *testing.M) {
	pool, err := dockertest.NewPool(dockerEndpoint)
	if err != nil {
		log.Fatalf("Could not connect to docker: %s", err)
	}

	resource, err := pool.Run("mysql", "5.7", []string{"MYSQL_ROOT_PASSWORD=secret"})
	if err != nil {
		log.Fatalf("Could not start resource: %s", err)
	}

	if err := pool.Retry(func() error {
		var err error
		db, err = sql.Open("mysql", fmt.Sprintf("root:secret@(localhost:%s)/mysql", resource.GetPort("3306/tcp")))
		if err != nil {
			return err
		}
		return db.Ping()
	}); err != nil {
		log.Fatalf("Could not connect to docker: %s", err)
	}

    code := m.Run()
    // Unfortunately you can't put this in a defer function because os.Exit terminates immediately.
    if err := pool.Purge(resource); err != nil {
        log.Fatalf("Could not purge resource: %s", err)
    }
	os.Exit(code)
}

func TestSomething(t *testing.T) {
	// db.Query()
}
```

### Setting up Travis-CI

You can run the Docker integration on Travis easily:

```yml
# Sudo is required for docker
sudo: required

# Enable docker
services:
  - docker
```

## Troubleshoot & FAQ

### Out of disk space

Try cleaning up the images with [docker-cleanup-volumes](https://github.com/chadoe/docker-cleanup-volumes).

### Removing old containers

Sometimes container clean up fails. Check out
[this stackoverflow question](http://stackoverflow.com/questions/21398087/how-to-delete-dockers-images) on how to fix this.

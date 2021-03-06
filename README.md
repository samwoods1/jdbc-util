[![Build Status](https://travis-ci.org/puppetlabs/jdbc-util.png?branch=master)](https://travis-ci.org/puppetlabs/jdbc-util)

# jdbc-util

_Being a collection of common jdbc utility code we use at Puppet
Labs._

## Usage

### Lazy connection pool

The pool namespace provides functions to create a hikaricp connection pool that
is lazily initialized witha user-provided init function, and which will not hand
out connections until that function has run.

```clojure
(let [timeout-ms-when-starting 5000
      ;; Accepts most hikari options, with clojure style names.
      options {:pool-name "app-pool-1"
               :jdbc-url "jdbc:postgresql:the_gibson"
               :username "eugene"
               :password "god"}
      init-fn (fn [db] (migrate-database db))
      db-pool (pool/connection-pool-with-delayed-init
               (pool/options->hikari-config options)
               init-fn
               timeout-ms-when-starting)]
  ;; ... wait a while, do some work ...
  (jdbc/query db-pool ["SELECT * FROM garbage_files;"]))
```

The database pool is wrapped in a future which loops attempting to connect to
the database. Once it connects, it runs `init-fn`, passing in the datasource in
a map as java.jdbc expects it (`{:datasource datasource}`). Once the init
function completes, the pool future is realized and it will hand out connections
transparently. If a connection is requested before the future is realized, it
will wait for the configured timeout (the third argument to
`connection-pool-with-delayed-init`) for it to become realized, and if it times
out raise a `java.sql.SQLTransientConnectionException`.

#### Options

Several functions are provided to configure the pool using either a database
spec (such as java.jdbc would use) or hikari options. See the docstrings for
`pool/spec->options` and `pool/options->hikari-config`. The
`pool/connection-pool-with-delayed-init` function expects a native
`HikariConfig` object.

#### Status

The pool also implements a status protocol (`pool/PoolStatus`) to check its
state and whether the initialization function was successful. The status method
will return a map representing the current pool health. Before the pool is
finished initializing the map will be:

```clojure
{:state :starting}
```

Once started the state becomes `:ready` or `:error`. When in an error state, an
additional `:messages` key will be present containing one or more error
messages. For example:

```clojure
{:state :error
 :messages ["Initialization resulted in an error: A rabbit's in the system."]}
```

When the initialization function throws an exception, the pool will continue to
function normally but the status will show an error state. The exception message
will appear under `:messages`, and the full exception object can be retrieved by
calling `init-error` on the pool.

The status is implemented using an io.dropwizard.metrics HealthCheckRegistry. If
one is provided to the hikari config under `:health-check-registry` it will be
used, otherwise one will be created automatically.

#### Lifetime

The pool implements a lifetime protocol (`pool/PoolLifetime`) to provide methods
of determining if the initialization is complete, cancelling the initialization
if possible, or waiting on the completion of the initialization.  Additionally
a new close interface `close-after-ready` waits to close the datasource until
the init routine is complete.

Before the initialization is complete `init-complete?` returns false.  If the
initialization routine finishes, fails or was cancelled, `init-complete?` returns
true.

`cancel-init` will attempt to cancel the async initialization process and return
true if it was cancelled, or false if it wasn't.  If the init routine has already
completed, `cancel-init` returns false.

`close-after-ready` has two forms, one that will block forever until the init
routine completes, or one that accepts a timeout in milliseconds.  If the
timeout is exceeded prior to the init completing, the routine will attempt to cancel
the init and then immediately close the connection.

## Support

To file a bug, please open a Jira ticket against this project. Bugs and PRs are
addressed on a best-effort basis. Puppet Inc. does not guarantee support for
this project.

## Running tests
You'll need PostgreSQL installed (9.4 is the mainly-used version for us right
now), and set up a "jdbc_util_test" DB and user with password "foobar" that has
the permission to create databases and users.

To give the "jdbc_util_test" user permission to create databases, open up a psql
session and then run:
```sql
ALTER ROLE jdbc_util_test CREATEDB CREATEROLE;
```

## License

Copyright © 2016 Puppet, Inc.

Distributed under the [Apache License, Version 2.0](http://www.apache.org/licenses/LICENSE-2.0.html)

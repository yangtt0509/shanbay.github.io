# Basic Usage
## Configuration

All you need to get you started is the configuration describing your database connections
and passing it to a ``DatabaseManager`` instance.

```
from orator import DatabaseManager

config = {
    'mysql': {
        'driver': 'mysql',
        'host': 'localhost',
        'database': 'database',
        'user': 'root',
        'password': '',
        'prefix': ''
    }
}

db = DatabaseManager(config)
```

If you have multiple databases configured you can specify which one is the default:

```
config = {
    'default': 'mysql',
    'mysql': {
        'driver': 'mysql',
        'host': 'localhost',
        'database': 'database',
        'user': 'root',
        'password': '',
        'prefix': ''
    }
}
```

>**Note**  
>
>Changed in version 0.8.0:
>
>For MySQL and PostgreSQL, the connectors are configured
to return unicode strings. If you want to use the previous behavior just set the
use_unicode configuration option to False.

## Read / Write connections

Sometimes you may wish to use one database connection for SELECT statements,
and another for INSERT, UPDATE, and DELETE statements. Orator makes this easy,
and the proper connections will always be used whether you use raw queries, the query
builder or the actual ORM

Here is an example of how read / write connections should be configured:

```
config = {
    'mysql': {
        'read': {
            'host': '192.168.1.1'
        },
        'write': {
            'host': '192.168.1.2'
        },
        'driver': 'mysql',
        'database': 'database',
        'username': 'root',
        'password': '',
        'prefix': ''
    }
}
```

Note that two keys have been added to the configuration dictionary: ``read`` and ``write``.
Both of these keys have dictionary values containing a single key: ``host``.
The rest of the database options for the ``read`` and ``write`` connections
will be merged from the main ``mysql`` dictionary. So, you only need to place items
in the ``read`` and ``write`` dictionaries if you wish to override the values in the main dictionary.
So, in this case, ``192.168.1.1`` will be used as the "read" connection, while ``192.168.1.2``
will be used as the "write" connection. The database credentials, prefix, character set,
and all other options in the main ``mysql`` dictionary will be shared across both connections.

## Running queries

Once you have configured your database connection, you can run queries.

>**Note**  
>
>Changed in version 0.9:
>
>The following examples use the qmark syntax for parameter binding.
In previous versions of Orator, you had to adapt it to the parameter style of
the underlying backend package. Now, you can set use_qmark to True in your
configuration to allow the qmark syntax for all packages, making Orator trully
database agnostic.
>
>This will be the default in the next major version.

### Running a select query

```
results = db.select('select * from users where id = ?', [1])
```

The ``select`` method will always return a list of results.

### Running an insert statement

```
db.insert('insert into users (id, name) values (?, ?)', [1, 'John'])
```

### Running an update statement

```
db.update('update users set votes = 100 where name = ?', ['John'])
```

### Running a delete statement

```
db.delete('delete from users')
```

>**Note**  
>
>The ``update`` and ``delete`` statements return the number of rows affected by the operation.

### Running a general statement

```
db.statement('drop table users')
``` 

## Database transactions

To run a set of operations within a database transaction, you can use the ``transaction`` method
which is a context manager:

```
with db.transaction():
    db.table('users').update({votes: 1})
    db.table('posts').delete()
```

Any exception thrown within a transaction block will cause the transaction to be rolled back
automatically.

Sometimes you may need to start a transaction yourself:

```
db.begin_transaction()
```

You can rollback a transaction with the ``rollback`` method:

```
db.rollback()
```

You can also commit a transaction via the ``commit`` method:

```
db.commit()
```

>**Warning**  
> 
>By default, all underlying DBAPI connections are set to be in autocommit mode meaning that you don't need to explicitly commit after each operation.

## Accessing connections

When using multiple connections, you can access them via the ``connection()`` method:

```
users = db.connection('foo').table('users').get()
```

You also can access the raw, underlying dbapi connection instance:

```
db.connection().get_connection()
```

Sometimes, you may need to reconnect to a given database:

```
db.reconnect('foo')
```

If you need to disconnect from the given database, use the ``disconnect`` method:

```
db.disconnect('foo')
```

## Query logging

Orator can log all queries that are executed.
By default, this is turned off to avoid unnecessary overhead, but if you want to activate it
you can either add a ``log_queries`` key to the config dictionary:

```
config = {
    'mysql': {
        'driver': 'mysql',
        'host': 'localhost',
        'database': 'database',
        'username': 'root',
        'password': '',
        'prefix': '',
        'log_queries': True
    }
}
```

or activate it later on:

```
db.connection().enable_query_log()
```

Now, the logger ``orator.connection.queries`` will be logging queries at **debug** level:

```
Executed SELECT COUNT(*) AS aggregate FROM "users" in 1.18ms

Executed INSERT INTO "users" ("email", "name", "updated_at") VALUES ('foo@bar.com', 'foo', '2015-04-01T22:59:25.810216'::timestamp) RETURNING "id" in 3.6ms
```

>**Note**  
>
>These log messages above are those logged for **MySQL** and **PostgreSQL** connections which support
>displaying full request sent to the database.
>For **SQLite** connections, the format is as follows:  
>    ```
>   Executed ('SELECT COUNT(*) AS aggregate FROM "users"', []) in 0.12ms
>    ```

### Customizing log messages

Each log record sent by the logger comes with the ``query`` and ``elapsed_time`` keywords so that
you can customize the log message:

```
import logging

logger = logging.getLogger('orator.connection.queries')
logger.setLevel(logging.DEBUG)

formatter = logging.Formatter(
    'It took %(elapsed_time)sms to execute the query %(query)s'
)

handler = logging.StreamHandler()
handler.setFormatter(formatter)

logger.addHandler(handler)
```
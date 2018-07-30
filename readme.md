# express-mysql-session-sharded

A MySQL session store for [express.js](http://expressjs.com/).

This is a fork of express-mysql-session https://github.com/chill117/express-mysql-session that allows for dynamic database switching for a 'sharded' set-up.

[![Build Status](https://travis-ci.org/chill117/express-mysql-session.svg?branch=master)](https://travis-ci.org/chill117/express-mysql-session) [![Status of Dependencies](https://david-dm.org/chill117/express-mysql-session.svg)](https://david-dm.org/chill117/express-mysql-session)


## Installation

For now express-mysql-session-sharded is not part of NPM so you will need to manually add the github repo url to your package.json file to install. Then run npm install from the command line. e.g:

"express-mysql-session-sharded": "https://github.com/TommyBs/express-mysql-session-sharded.git",

## Important Notes

There are some potential gotchas. At the moment you will need to create the `sessions` database table on all your relevant DB machines. Also note that only the `get`, `set`, `touch` and `delete` session methods implement switching databases. So you will need to either use MySQL event scheduler to remove invalid sessions or use a cronjob that calls a script to do so. 

At the moment you also still need to pass in a standard connection as well.

### Older Versions

For users who are still using express-mysql-session `0.x`. Changes have been made to the constructor, which are backwards compatible, but you could run into troubles if using an older version of this module with the latest documentation. You can find the documentation for the older version [here](https://github.com/chill117/express-mysql-session/tree/9fbcf51416a00a7a525c1e6e431033125a2945b0).

### Session Table Collation

This module creates a database table to save session data. This data is stored in a MySQL text field with the [utf8mb4](https://dev.mysql.com/doc/refman/5.5/en/charset-unicode-utf8mb4.html) collation - added in [MySQL 5.5.3](https://dev.mysql.com/doc/relnotes/mysql/5.5/en/news-5-5-3.html). The reason for this is to fully support the utf8 character set. If you absolutely must use an older version of MySQL, create your sessions table before initializing the `MySQLStore`.


## How to Use

Use with your express session middleware, like this (NOTE THE NEW `shardFunction` option. This is a function that gets passed the session_id as a single parameter, you should use this to work out what shard the session sits on. In the example below it uses a simple md5 -> int -> modulus conversion to work out the shard_id. `DB` in this instance would be a helper class that determines what database pool to use:
```js
var express = require('express');
var app = module.exports = express();
var session = require('express-session');
var MySQLStore = require('express-mysql-session-sharded')(session);

var options = {
	host: 'localhost',
	port: 3306,
	user: 'session_test',
	password: 'password',
	database: 'session_test',
	shardFunction:function(session_id) {
  	let hash = crypto.createHash('md5');
	let shard_id = hash.update(session_id).digest('hex');
	shard_id = (parseInt(shard_id.replace(/[^\d]+/ig, '')) % 1024) + 1;
  	return DB.getPoolForShardId(shard_id)
	}
};

var sessionStore = new MySQLStore(options);

app.use(session({
	key: 'session_cookie_name',
	secret: 'session_cookie_secret',
	store: sessionStore,
	resave: false,
	saveUninitialized: false
}));
```

The session store will internally create a `mysql` [connection pool](https://github.com/mysqljs/mysql#pooling-connections) which handles the (re)connection to the database. By default, the pool consists of 1 connection, but you can override this using the `connectionLimit` option. There are additional [pool options](https://github.com/mysqljs/mysql#pool-options) you can provide, which will be passed to the constructor of the `mysql` connection pool.

The sessions database table should be automatically created, when using default options. If for whatever reason the table is not created, you can find the schema [here](https://github.com/chill117/express-mysql-session/blob/master/schema.sql).

### With an existing MySQL connection or pool

To pass in an existing MySQL database connection or pool, you would do something like this:
```js
var mysql = require('mysql');
var session = require('express-session');
var MySQLStore = require('express-mysql-session')(session);

var options = {
    host: 'localhost',
    port: 3306,
    user: 'db_user',
    password: 'password',
    database: 'db_name'
};

var connection = mysql.createConnection(options); // or mysql.createPool(options);
var sessionStore = new MySQLStore({}/* session store options */, connection);
```

### Closing the session store

To cleanly close the session store:
```js
sessionStore.close();
```


### Options

Here is a list of all available options:
```js
var options = {
	// Host name for database connection:
	host: 'localhost',
	// Port number for database connection:
	port: 3306,
	// Database user:
	user: 'session_test',
	// Password for the above database user:
	password: 'password',
	// Database name:
	database: 'session_test',
	// Whether or not to automatically check for and clear expired sessions:
	clearExpired: true,
	// How frequently expired sessions will be cleared; milliseconds:
	checkExpirationInterval: 900000,
	// The maximum age of a valid session; milliseconds:
	expiration: 86400000,
	// Whether or not to create the sessions database table, if one does not already exist:
	createDatabaseTable: true,
	// Number of connections when creating a connection pool:
	connectionLimit: 1,
	// Whether or not to end the database connection when the store is closed.
	// The default value of this option depends on whether or not a connection was passed to the constructor.
	// If a connection object is passed to the constructor, the default value for this option is false.
	endConnectionOnClose: true,
	charset: 'utf8mb4_bin',
	schema: {
		tableName: 'sessions',
		columnNames: {
			session_id: 'session_id',
			expires: 'expires',
			data: 'data'
		}
	}
};
```


#### Configurable sessions table and column names

You can override the default sessions database table name and column names via the `schema` option:

```js
var session = require('express-session');
var MySQLStore = require('express-mysql-session')(session);

var options = {
	host: 'localhost',
	port: 3306,
	user: 'session_test',
	password: 'password',
	database: 'session_test',
	schema: {
		tableName: 'custom_sessions_table_name',
		columnNames: {
			session_id: 'custom_session_id',
			expires: 'custom_expires_column_name',
			data: 'custom_data_column_name'
		}
	}
};

var sessionStore = new MySQLStore(options);
```


### Debugging

`express-mysql-session` uses the [debug module](https://github.com/visionmedia/debug) to output debug messages to the console. To output all debug messages, run your node app with the `DEBUG` environment variable:
```
DEBUG=express-mysql-session* node your-app.js
```
This will output log messages as well as error messages from `express-mysql-session`.

If you also might need MySQL-related debug and error messages, see [debugging node-mysql](https://github.com/mysqljs/mysql#debugging-and-reporting-problems).


## Contributing

There are a number of ways you can contribute:

* **Improve or correct the documentation** - All the documentation is in this readme file. If you see a mistake, or think something should be clarified or expanded upon, please [submit a pull request](https://github.com/chill117/express-mysql-session/pulls/new)
* **Report a bug** - Please review [existing issues](https://github.com/chill117/express-mysql-session/issues) before submitting a new one; to avoid duplicates. If you can't find an issue that relates to the bug you've found, please [create a new one](https://github.com/chill117/express-mysql-session/issues).
* **Request a feature** - Again, please review the [existing issues](https://github.com/chill117/express-mysql-session/issues) before posting a feature request. If you can't find an existing one that covers your feature idea, please [create a new one](https://github.com/chill117/express-mysql-session/issues).
* **Fix a bug** - Have a look at the [existing issues](https://github.com/chill117/express-mysql-session/issues) for the project. If there's a bug in there that you'd like to tackle, please feel free to do so. I would ask that when fixing a bug, that you first create a failing test that proves the bug. Then to fix the bug, make the test pass. This should hopefully ensure that the bug never creeps into the project again. After you've done all that, you can [submit a pull request](https://github.com/chill117/express-mysql-session/pulls/new) with your changes.

Before you contribute code, please read through at least some of the source code for the project. I would appreciate it if any pull requests for source code changes follow the coding style of the rest of the project.

Now if you're still interested, you'll need to get your local environment configured.


### Configure Local Environment

#### Step 1: Get the Code

First, you'll need to pull down the code from GitHub:
```
git clone https://github.com/chill117/express-mysql-session.git
```

#### Step 2: Install Dependencies

Second, you'll need to install the project dependencies as well as the dev dependencies. To do this, simply run the following from the directory you created in step 1:
```
npm install
```

#### Step 3: Set Up the Test Database

Now, you'll need to set up a local test database:
```js
{
	host: 'localhost',
	port: 3306,
	user: 'session_test',
	password: 'password',
	database: 'session_test'
};
```
*The test database settings are located in [test/config.js](https://github.com/chill117/express-mysql-session/blob/master/test/config.js)*

Alternatively, you can provide custom database configurations via environment variables:
```
DB_HOST="localhost"
DB_PORT="3306"
DB_USER="session_test"
DB_PASS="password"
DB_NAME="session_test"
```


### Running Tests

With your local environment configured, running tests is as simple as:
```
npm test
```


## Changelog

See [changelog.md](https://github.com/chill117/express-mysql-session/blob/master/changelog.md)

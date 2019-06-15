# Starting Logux Server Project

In this guide we will create the basic server to the most simple case:

* You have simple HTTP server to serve HTML and static CSS and JS files.
* You use [Logux Redux] or [Logux Client] on the client side.
* Logux server do most of back-end business logic.
* You write Logux server on Node.js.

If you want to use another language for server check [Logux Proxy] section.

[Logux Client]: ./5-creating-client.md
[Logux Redux]: ./3-creating-redux.md
[Logux Proxy]: ./2-creating-proxy.md


## Create the Project

First you need to [install Node.js] (version 10.0 or later).

Create a directory with a project. We will use `project-logux`, but you can
replace it to more relevant to your project.

```sh
mkdir project-logux
cd project-logux
```

Create `package.json` with:

```json
{
  "name": "project-logux",
  "private": true,
  "main": "index.js"
}
```

<details open><summary><b>npm</b></summary>

```sh
npm i @logux/server
```

</details>
<details><summary><b>yarn</b></summary>

```sh
yarn add @logux/server
```

</details>

Create `index.js` with:

```js
const { Server } = require('@logux/server')

const server = new Server(
  Server.loadOptions(process, {
    subprotocol: '0.1.0',
    supports: '^0.1.0',
    root: __dirname
  })
)

server.auth((userId, token) => {
  return false
})

server.listen()
```

The simple Logux server is ready. You can start it with:

```sh
node index.js
```

To stop the server press `Command`+`.` on Mac OS X and `Ctrl`+`C` on Linux
and Windows.

[install Node.js]: https://nodejs.org/en/download/package-manager/

## Database

Logux Server can work with any database. We will use PostgreSQL only as example.

Install PostgreSQL and its tools for Node.js:

<details open><summary><b>npm</b></summary>

```sh
npm i dotenv pg-promise node-pg-migrate pg
```

</details>
<details><summary><b>yarn</b></summary>

```sh
yarn add dotenv pg-promise node-pg-migrate pg
```

</details>

Create database. Use [this advice] on `role does not exist` error.

```sh
createdb project-logux
```

Create `.env` config with URL to your database. Put this file to `.gitignore`.

```
DATABASE_URL=postgres://localhost/project-logux
```

Create new database schema migration:

```sh
npx node-pg-migrate create create_users
```

Open generated file from `migrations/` and create `users` table:

```js
exports.up = pgm => {
  pgm.createExtension('pgcrypto')
  pgm.createTable('users', {
    email: { type: 'varchar', notNull: true, unique: true },
    token: { type: 'varchar', default: pgm.func('gen_random_uuid') },
    password: { type: 'varchar', notNull: true }
  })
}

exports.down = pgm => {
  pgm.dropExtension('pgcrypto')
  pgm.dropTable('users')
}
```

Run migration:

```sh
npx node-pg-migrate up
```

Connect to database in the server:

```diff
  const { Server } = require('@logux/server')
+ const pg = require('pg-promise')

  const server = new Server(
    Server.loadOptions(process, {
      subprotocol: '0.1.0',
      supports: '^0.1.0',
      root: __dirname
    })
  )

+ let db = pg()(process.env.DATABASE_URL)
```

[Install PostgreSQL]: https://www.postgresql.org/download/
[this advice]: https://stackoverflow.com/questions/16973018/createuser-could-not-connect-to-database-postgres-fatal-role-tom-does-not-e


## Authentication

Logux Server uses user ID (we will use email as ID) and token to authenticate
user. We can’t use password on every connection, because it is unsafe to
keep password in the client memory.

In this example, Logux connects as `guest`, sends email and password
by `login` action to get token and save it in the memory.
Then client will reconnect with it’s own email and token.

Replace `server.auth(…)` with this code:

```js
// Small helper to return user from database
function byEmail (email) {
  return db.one('SELECT * FROM users WHERE email = ?', email)
}

server.auth(async (userId, token) => {
  if (userId === 'guest') {
    // Guests don’t need any validations
    return true
  } else {
    let user = await byEmail(userId)
    return user.token === token
  }
})

// Handler for { type: 'login', email, password } actions
server.type('login', {
  async access (ctx) {
    // This action is accepted only from guests
    return ctx.userId === 'guest'
  },
  async process (ctx, action) {
    let user = await byEmail(action.email)
    if (user && user.password === action.password) {
      ctx.sendBack({ type: 'login/done', token })
    } else {
      ctx.sendBack({ type: 'login/error' })
    }
  }
})
```

**[Next chapter →](./3-creating-redux.md)**
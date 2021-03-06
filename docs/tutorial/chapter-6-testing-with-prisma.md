# Chapter 6 <br> Testing With Prisma {docsify-ignore}

There's a couple of things you'll have to do in order to run integration tests against your API now that it's connected to a real development database. Please note that a lot of the following steps will most likely be simplified in the future. We're just not there yet. In this chapter, you'll learn about:

- Custom Jest environment
- Integration test with a real database

## How does it work?

To perform integration testing against a real database, here are the high level steps we will follow _for every tests_:

- Connect to a Postgres database. Most likely your dev database.
- Migrate our database schema to a randomly generated schema of that database. This ensures that every tests runs from a clean un-seeded database
- Make the Prisma Client connect to that Postgres schema
- Run your test
- Teardown the schema entirely

## Setting up the environment

To achieve some of the steps described above, we'll use a custom Jest environment.

Create a `tests/nexus-test-environment.js` module and copy & paste the following to it

```js
// tests/nexus-test-environment.js
const { Client } = require('pg')
const NodeEnvironment = require('jest-environment-node')
const { nanoid } = require('nanoid')
const util = require('util')
const exec = util.promisify(require('child_process').exec)

const prismaBinary = './node_modules/.bin/prisma'

/**
 * Custom test environment for Nexus, Prisma and Postgres
 */
class PrismaTestEnvironment extends NodeEnvironment {
  constructor(config) {
    super(config)

    // Generate a unique schema identifier for this test context
    this.schema = `test_${nanoid()}`

    // Generate the pg connection string for the test schema
    this.databaseUrl = `postgres://postgres:postgres@localhost:5432/testing?schema=${this.schema}`
  }

  async setup() {
    // Set the required environment variable to contain the connection string
    // to our database test schema
    process.env.DATABASE_URL = this.databaseUrl
    this.global.process.env.DATABASE_URL = this.databaseUrl

    // Run the migrations to ensure our schema has the required structure
    await exec(`${prismaBinary} migrate up --create-db --experimental`)

    return super.setup()
  }

  async teardown() {
    // Drop the schema after the tests have completed
    const client = new Client({
      connectionString: this.databaseUrl,
    })
    await client.connect()
    await client.query(`DROP SCHEMA IF EXISTS "${this.schema}" CASCADE`)
    await client.end()
  }
}

module.exports = PrismaTestEnvironment
```

Make sure that the `databaseUrl` property has the right credentials to connect to your own database.
Leave the `/testing?schema=...` part though. This ensures that your tests will add data to your Postgres instance in a separate database called `testing` in a schema that randomly generated.

Then, configure Jest to use that custom environment in your `package.json`

```diff
"jest": {
  "preset": "ts-jest",
- "testEnvironment": "node",
+ "testEnvironment": "./tests/nexus-test-environment.js"
}
```

Finally, thanks to the `nexus-plugin-prisma`, the test context that we previously used should now be augmented with a `ctx.app.db` property. That db property holds an instance of the Prisma Client to give you access to the underlying testing database. This is useful, for instance, to seed your database or make sure that some data was properly inserted.

The last thing we need to do to setup our environment is to make sure that we properly close the database connection after all tests. To do that, head to your `tests/__helpers.ts` module and add the following

```diff
import {
  createTestContext as originalCreateTestContext,
  TestContext,
} from 'nexus/testing';

export function createTestContext(): TestContext {
  let ctx = {} as TestContext;

  beforeAll(async () => {
    Object.assign(ctx, await originalCreateTestContext());

    await ctx.app.start();
  });

  afterAll(async () => {
+   await ctx.app.db.client.disconnect();
    await ctx.app.stop();
  });

  return ctx;
}
```

## Updating our test

We're ready to update our test so that it uses our database. Wait though. Is there even something to change?
No, absolutely nothing. In fact, you can already try running Jest again and your test should pass. That's precisely the point of integration tests.

There's one thing we can do though. If you remember our previous test, the only part we couldn't test was whether or not the data had properly been persisted into the database.

Let's use the `ctx.app.db` property to fetch our database right after we've published the draft to ensure that it's been created by snapshotting the result.

```diff
// tests/Post.test.ts

it('ensures that a draft can be created and published', async () => {
  // ...

  // Publish the previously created draft
  const publishResult = await ctx.app.query(
    `
    mutation publishDraft($draftId: Int!) {
      publish(draftId: $draftId) {
        id
        title
        body
        published
      }
    }
  `,
    { draftId: draftResult.createDraft.id }
  )

  // Snapshot the published draft and expect `published` to be true
  expect(publishResult).toMatchInlineSnapshot(`
    Object {
      "publish": Object {
        "body": "...",
        "id": 1,
        "published": true,
        "title": "Nexus",
      },
    }
  `)

+ const persistedData = await ctx.app.db.client.post.findMany()

+ expect(persistedData).toMatchInlineSnapshot()
})
```

The new snapshot should look like the following. It proves that our database did persist that data and that we have exactly one item in it.

```diff
expect(persistedData).toMatchInlineSnapshot(`
+   Array [
+     Object {
+       "body": "...",
+       "id": 1,
+       "published": true,
+       "title": "Nexus",
+     },
+   ]
  `)
```

## Wrapping up

Congrats, you've performed your first real-world integration test. The fact that integration tests are completely decoupled from the implementation of your GraphQL API makes it a lot easier to maintain your test suite as you evolve your API. What matters is only the data that it produces, which also helps you cover your app a lot more than a single unit test.

About our app, it's starting to take shape but it's still lacking something pretty important in any application: authentication.

Come onto the next chapter to get that added to your app!

<div class="NextIs NextChapter"></div>

[➳](/tutorial/chapter-7-authentication)

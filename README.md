# GraphQL Server with Authentication & Permissions

This example shows how to implement a **GraphQL server with TypeScript** with the following stack:

- [**Apollo Server**](https://github.com/apollographql/apollo-server): HTTP server for GraphQL APIs
- [**GraphQL Nexus**](https://nexusjs.org/docs/): GraphQL schema definition and resolver implementation
- [**GraphQL Shield**](https://github.com/maticzav/graphql-shield): Authorization/permission layer for GraphQL schemas
- [**Prisma Client**](https://www.prisma.io/docs/concepts/components/prisma-client): Databases access (ORM)
- [**Prisma Migrate**](https://www.prisma.io/docs/concepts/components/prisma-migrate): Database migrations
- [**PostgreSQL**](https://www.postgresql.org/): Local, file-based SQL database

## Contents

- [Getting Started](#getting-started)
- [Using the GraphQL API](#using-the-graphql-api)
- [Evolving the app](#evolving-the-app)
- [Switch to another database (e.g. MySQL, SQL Server)](#switch-to-another-database-eg-postgresql-mysql-sql-server)
- [Next steps](#next-steps)

## Getting started

### 1. Download example and install dependencies

Clone this repository:

```
git clone https://github.com/Andrei-R1/GraphQL-TS-Boilerplate
```

Install npm dependencies:

```
npm install
```

### 2. Create and seed the database

Run the following command to create your PostgreSQL database file.

```
npx prisma migrate dev
```

When `npx prisma migrate dev` is executed against a newly created database.

### 3. Start the GraphQL server

Launch your GraphQL server with this command:

```
npm run dev
```

Navigate to [http://localhost:4000](http://localhost:4000) in your browser to explore the API of your GraphQL server in a [GraphQL Playground](https://github.com/prisma/graphql-playground).

## Using the GraphQL API

The schema that specifies the API operations of your GraphQL server is defined in [`./schema.graphql`](./schema.graphql). Below are a number of operations that you can send to the API using the GraphQL Playground.

Feel free to adjust any operation by adding or removing fields. The GraphQL Playground helps you with its auto-completion and query validation features.

### Register a new user

You can send the following mutation in the Playground to sign up a new user and retrieve an authentication token for them:

```graphql
mutation {
  signup(name: "Sarah", email: "sarah@prisma.io", password: "HelloWorld42") {
    token
  }
}
```

### Log in an existing user

This mutation will log in an existing user by requesting a new authentication token for them.

```graphql
mutation {
  login(email: "sarah@prisma.io", password: "HelloWorld42") {
    token
  }
}
```

### Check whether a user is currently logged in with the `me` query

For this query, you need to make sure a valid authentication token is sent along with the `Bearer`-prefix in the `Authorization` header of the request:

```json
{
  "Authorization": "Bearer __YOUR_TOKEN__"
}
```

With a real token, this looks similar to this:

```json
{
  "Authorization": "Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VySWQiOiJjanAydHJyczFmczE1MGEwM3kxaWl6c285IiwiaWF0IjoxNTQzNTA5NjY1fQ.Vx6ad6DuXA0FSQVyaIngOHYVzjKwbwq45flQslnqX04"
}
```

Inside the Playground, you can set HTTP headers in the bottom-left corner:

![](https://imgur.com/ToRcCTj.png)

Once you've set the header, you can send the following query to check whether the token is valid:

```graphql
{
  me {
    id
    name
    email
  }
}
```

### Authenticating GraphQL requests

In this example, you authenticate your GraphQL requests using the `Authorization` header field of the HTTP requests which are sent from clients to your GraphQL server. The required authentication token is returned by successful `signup` and `login` mutations.

Using the GraphQL Playground, the `Authorization` header can be configured in the **HTTP HEADERS** tab in the bottom-left corner of the GraphQL Playground. The values for the HTTP headers are defined in JSON format. Note that the authentication token needs to be sent with the `Bearer `-prefix:

```json
{
  "Authorization": "Bearer __YOUR_TOKEN__"
}
```

With a "real" authentication token, it looks similar to this:

```json
{
  "Authorization": "Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VySWQiOiJjanAydHJyczFmczE1MGEwM3kxaWl6c285IiwiaWF0IjoxNTQzNTA5NjY1fQ.Vx6ad6DuXA0FSQVyaIngOHYVzjKwbwq45flQslnqX04"
}
```

As mentioned before, you can set HTTP headers in the bottom-left corner of the GraphQL Playground:

![img](https://imgur.com/ToRcCTj.png)

## Evolving the app

Evolving the application typically requires two steps:

1. Migrate your database using Prisma Migrate
2. Update your application code

For the following example scenario, assume you want to add a "profile" feature to the app where users can create a profile and write a short bio about themselves.

### 1. Migrate your database using Prisma Migrate

The first step is to add a new table, e.g. called `Profile`, to the database. You can do this by adding a new model to your [Prisma schema file](./prisma/schema.prisma) file and then running a migration afterwards:

```diff
// ./prisma/schema.prisma

model User {
  id      Int      @default(autoincrement()) @id
  name    String?
  email   String   @unique
+ profile Profile?
}

+model Profile {
+  id     Int     @default(autoincrement()) @id
+  bio    String?
+  user   User    @relation(fields: [userId], references: [id])
+  userId Int     @unique
+}
```

Once you've updated your data model, you can execute the changes against your database with the following command:

```
npx prisma migrate dev --name add-profile
```

This adds another migration to the `prisma/migrations` directory and creates the new `Profile` table in the database.

### 2. Update your application code

You can now use your `PrismaClient` instance to perform operations against the new `Profile` table. Those operations can be used to implement queries and mutations in the GraphQL API.

#### 2.1. Add the `Profile` type to your GraphQL schema

First, add a new GraphQL type via Nexus' `objectType` function:

```diff
// ./src/schema.ts

+const Profile = objectType({
+  name: 'Profile',
+  definition(t) {
+    t.nonNull.int('id')
+    t.string('bio')
+    t.field('user', {
+      type: 'User',
+      resolve: (parent, _, context) => {
+        return context.prisma.profile
+          .findUnique({
+            where: { id: parent.id || undefined },
+          })
+          .user()
+      },
+    })
+  },
+})

const User = objectType({
  name: 'User',
  definition(t) {
    t.nonNull.int('id');
    t.string('name');
    t.nonNull.string('email');
+    t.field('profile', {
+      type: 'Profile',
+      resolve: (parent, _, context) => {
+        return context.prisma.user
+          .findUnique({ 
+            where: { id: parent.id }, 
+          })
+          .profile();
+      },
+    });
  },
});
```

Don't forget to include the new type in the `types` array that's passed to `makeSchema`:

```diff
export const schema = makeSchema({
  types: [
    Query,
    Mutation,
    User,
+   Profile,
    DateTime,
  ],
  // ... as before
}
```

Note that in order to resolve any type errors, your development server needs to be running so that the Nexus types can be generated. If it's not running, you can start it with `npm run dev`.

#### 2.2. Add a `createProfile` GraphQL mutation

```diff
// ./src/schema.ts

const Mutation = objectType({
  name: 'Mutation',
  definition(t) {

    // other mutations

+   t.field('addProfileForUser', {
+     type: 'Profile',
+     args: {
+       userUniqueInput: nonNull(
+         arg({
+           type: 'UserUniqueInput',
+         }),
+       ),
+       bio: stringArg()
+     }, 
+     resolve: async (_, args, context) => {
+       return context.prisma.profile.create({
+         data: {
+           bio: args.bio,
+           user: {
+             connect: {
+               id: args.userUniqueInput.id || undefined,
+               email: args.userUniqueInput.email || undefined,
+             }
+           }
+         }
+       })
+     }
+   })

  }
})
```

Finally, you can test the new mutation like this:

```graphql
mutation {
  addProfileForUser(
    userUniqueInput: {
      email: "mahmoud@prisma.io"
    }
    bio: "I like turtles"
  ) {
    id
    bio
    user {
      id
      name
    }
  }
}
```

<details><summary>Expand to view more sample Prisma Client queries on <code>Profile</code></summary>

Here are some more sample Prisma Client queries on the new `<code>`Profile `</code>` model:

##### Create a new profile for an existing user

```ts
const profile = await prisma.profile.create({
  data: {
    bio: 'Hello World',
    user: {
      connect: { email: 'alice@prisma.io' },
    },
  },
})
```

##### Create a new user with a new profile

```ts
const user = await prisma.user.create({
  data: {
    email: 'john@prisma.io',
    name: 'John',
    profile: {
      create: {
        bio: 'Hello World',
      },
    },
  },
})
```

##### Update the profile of an existing user

```ts
const userWithUpdatedProfile = await prisma.user.update({
  where: { email: 'alice@prisma.io' },
  data: {
    profile: {
      update: {
        bio: 'Hello Friends',
      },
    },
  },
})
```

</details>

## Switch to another database (e.g. MySQL, SQL Server, MongoDB)

If you want to try this example with another database than SQLite, you can adjust the the database connection in [`prisma/schema.prisma`](./prisma/schema.prisma) by reconfiguring the `datasource` block.

Learn more about the different connection configurations in the [docs](https://www.prisma.io/docs/reference/database-reference/connection-urls).

<details><summary>Expand for an overview of example configurations with different databases</summary>

### MySQL

For MySQL, the connection URL has the following structure:

```prisma
datasource db {
  provider = "mysql"
  url      = "mysql://USER:PASSWORD@HOST:PORT/DATABASE"
}
```

Here is an example connection string with a local MySQL database:

```prisma
datasource db {
  provider = "mysql"
  url      = "mysql://janedoe:mypassword@localhost:3306/notesapi"
}
```

### Microsoft SQL Server

Here is an example connection string with a local Microsoft SQL Server database:

```prisma
datasource db {
  provider = "sqlserver"
  url      = "sqlserver://localhost:1433;initial catalog=sample;user=sa;password=mypassword;"
}
```

### MongoDB

Here is an example connection string with a local MongoDB database:

```prisma
datasource db {
  provider = "mongodb"
  url      = "mongodb://USERNAME:PASSWORD@HOST/DATABASE?authSource=admin&retryWrites=true&w=majority"
}
```

Because MongoDB is currently in [Preview](https://www.prisma.io/docs/about/releases#preview), you need to specify the `previewFeatures` on your `generator` block:

```
generator client {
  provider        = "prisma-client-js"
  previewFeatures = ["mongodb"]
}
```

</details>

## Next steps

- Check out the [Prisma docs](https://www.prisma.io/docs)
- Share your feedback in the [`prisma2`](https://prisma.slack.com/messages/CKQTGR6T0/) channel on the [Prisma Slack](https://slack.prisma.io/)
- Create issues and ask questions on [GitHub](https://github.com/prisma/prisma/)
- Watch our biweekly "What's new in Prisma" livestreams on [Youtube](https://www.youtube.com/channel/UCptAHlN1gdwD89tFM3ENb6w)

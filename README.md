Storex is a minimal storage layer as a foundation for easing common problems around storing and moving data around. Allowing you to describe your data layout as a graph and providing different plugins, it helps you interact with (No)SQL databases, data migration, offline first applications architecture, creating and consuming REST/GraphQL APIs, permission management, finding optimization opportunaties and more. The aim is to provide a minimalistic common ground/language for working with your data, providing packages for solving the most common problems around data, while giving you easy access to the underlying machinery to do the things that are specific to your application. Everything together that means that every problem you encounter while rapidly iterating towards a serious product, from choosing a suitable DB to suddenly realizing you need to migrate your data model, or even switch DBs, will get a ton easier because you don't have to solve them for the 109th time yourself.

**Status:** Proof of concept used in production to interact with IndexedDB while having the freedom to shift to the cloud and decentralize storage in the near future. Needs a lot more development, but implemented functionality is working. That being said, the API is subject to change between minor versions until the 1.0 release. Please consider contributing through easy to pick up tasks to get you started!

Installation
============

Storex is a collection of Node.js modules (written in TypeScript) available through NPM, meant to be used both client- and server-side. To start, you need the core and a backend:
```
$ npm install storex --save

$ # For a client-side DB
$ npm install storex-backend-dexie --save # IndexedDB through Dexie library

$ # For a server-side SQL DB
$ npm install storex-backend-sequelize --save # MySQL, PostgreSQL, SQLite, MSSQL through Sequelize

```

Basic usage
===========

First, configure a StorageBackend and set up the StorageManager, which will be the main point of access to define, query and manipulate your data. For more in-depth information on how to do all of this, please refer to [the docs](./docs/0-start-here.md).

```
import StorageManager from 'storex'
import { DexieStorageBackend } from 'storex-backend-dexie'

const storageBackend = new DexieStorageBackend({dbName: 'my-awesome-product'})
const storageManager = new StorageManager({ backend: storageBackend })
storageManager.registry.registerCollections({
    user: {
        version: new Date(2018, 11, 11),
        fields: {
            identifier: { type: 'string' },
            isActive: { type: 'boolean' },
        },
        indices: [
            { field: 'identifier' },
        ]
    },
    todoList: {
        version: new Date(2018, 7, 11),
        fields: {
            title: { type: 'string' },
        },
        relationships: [
            {childOf: 'user'} # creates one-to-many relationship
        ],
        indices: []
    },
    todoListEntry: {
        version: new Date(2018, 7, 11),
        fields: {
            content: {type: 'text'},
            done: {type: 'boolean'}
        },
        relationships: [
            {childOf: 'todoList', reverseAlias: 'entries'}
        ]
    }
})
await storageManager.finishInitialization()

const user = await storageManager.collection('user').createObject({
    identifier: 'email:boo@example.com',
    isActive: true,
    todoLists: [{
        title: 'Procrastinate this as much as possible',
        entries: [
            {content: 'Write intro article', done: true},
            {content: 'Write docs', done: false},
            {content: 'Publish article', done: false},
        ]
    }]
})
# user now contains things generated by underlying backend, like ids and random keys if you have such fields
console.log(user.id)

await storageManager.collection('todoList').findObjects({user: user.id}) # You can also use MongoDB-like queries
```

Further documentation
=====================

You can [find the docs here](./docs/0-start-here.md). Also, we'll be writing more and more automated tests which also serve as documentation.

Status and future development
=============================

At present, these features are implemented and tested:

- **One DB abstraction layer for client- and server-side applications:** Using Dexie for IndexedDB, or Sequelize for SQL databases. This allows you to write storage-related business logic portable between front- and back-end, while easily switching to non-SQL storage back-ends later if you so desire.
- **Defining data in a DB-agnostic way as a graph of collections**: By registering your data collections with the StorageManager, you can have an easily introspectable representation of your data model
- **Automatic creation of relationships in DB-agnostic way**: One-to-one, one-to-many and many-to-many relationships declared in DB-agnostic ways are automatically being taken care of by the underlying StorageBackend on creation.
- **MongoDB-style querying:** The .findObjects() and .findOneObject() methods of a collection take MongoDB-style queries, which will then be translated by the underlying StorageBackend.
- **Client-side full-text search using Dexie backend:** By passing a stemmer into the `DexieStorageBackend({stemmer: (text : string) => Promise<string[]>})` you can full-text search text fields using the fastest client-side full-text search engine yet!
- **Run automated storage-related tests in memory:** Using the Dexie back-end, you can pass in a fake IndexedDB implementation to run your storage in-memory for faster automated and manual testing.
- **Version management of data models:** For each collection, you can pass in an array of different date-versioned collection versions, and you'll be able to iterate over your data model versions through time.

The following items are on the roadmap in no particular order:

- [DB-agnostic data migrations](https://github.com/WorldBrain/storex/issues/3): An easy and unified way of doing data-level migrations if your data model changes, like providing defaults for new non-optional fields, splitting and merging fields, splitting and joing collections, etc.
- [Relationship fetching & filtering](https://github.com/WorldBrain/storex/issues/4): This would allow passing in an extra option to find(One)Object(s) signalling the back-end to also fetch relationship, which would translate to JOINs in SQL databases and use other configurable methods in other kinds of databases. Also, you could filter by relationships, like `collection('user').findObjects({'email.active': true})`.
- [API server and consumer](https://github.com/WorldBrain/storex/issues/5): Allows you to start developing your application fully-client side for rapid iteration, and move the storage to the cloud when you're ready wit greatly reduced effort.
- [Unified access control definition](https://github.com/WorldBrain/storex/issues/6): Define the rules of who can read/write what data, which can be enforced by your API server or a Backend as a Service like Firebase.
- **Field types for handling user uploads:** Allowing you to reference user uploads in your data-model, while choosing your own back-end to host them.
- **A caching layer:** Allows you to cache certain explicitly-configured queries in stores like Memcache and Redis
- **Synching back-end for offline-first applications:** An aggragente back-end which intelligently writes to a client-side database first, and syncs with the server when possible.
- **Composite back-end writing to multiple back-ends at once:** When you're switching databases or cloud providers, there may be period where your application needs to the exact same data to multiple database systems at once.
- **Assisting migrations from one database to another:** Creating standard procedures allowing copying data from one database to another with any paradigm translations that might be needed.
- **Server-side full-text search server integration":** Allow for example to store your data in MondoDB, but your full-text index in ElasticSearch.
- **Pre-compiled queries:** Let the backend know which kind of queries you're going to do, so the backend can optimize compilation, and you get a unified overview of how your applcation queries and manipulates data.
- **Query analytics:** Report query performance and production usage patterns to your anaylics backend to give you insight into possible optimization opportunities (such as what kind of indices to create.)

Also, Storex was built with decentralization in mind. The first available backend is Dexie, which allows you to user data on the client side. In the future, we see it possible to create backends for decentralized systems like [DAT](https://datproject.org/) to ease the transition and integration between centralized and decentralized back-ends as easy as possible.

Contributing
============

There are different ways you can contribute:

- **Report bugs:** The most simple way. Scream at us  :)  If you can find the time though, we'd really appreciate it if you could attach a PR with a failing unit test.
- **Tackle outstanding bugs:** Great way to dip your toes in the water. Some of the reported bugs may already have failing unit tests attached to them!
- **Propose new features:** Open an issue to describe a new feature you'd like to see. Please take the time to explain your reasoning behind the request, how it would benefit the Storex ecosystem and things like use-cases if relevant.
- **Implement new features:** We try our best to make extensive descriptions of how features should work, and a clear decision process if the features are yet to be designed. Features to be implemented [can be found here](https://github.com/WorldBrain/storex/issues?q=is%3Aissue+is%3Aopen+label%3Aenhancement) with help how to pick up those features.

### Building for development

Storex is written in [Typescript](https://www.typescriptlang.org/). As such, there's a lot of type information serving as documentation for the reader of the code. To start writing code, do the following:

1) Check out the Storex repo: `git clone git@github.com:WorldBrain/storex.git`. Skip steps 3, 4 and 5 if you're going to work on the Storex core package.
2) Run `yarn` inside the checked out repo.
3) Inside the checked out repo, run `npm link`.
4) In another directory, check a out back-end you'd like to work on: `git clone git@github.com:WorldBrain/storex-backend-dexie.git`
5) Run `yarn` again inside the checked out repo.
6) Run `yarn test:mocha:watch` to start hacking!

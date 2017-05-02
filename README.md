# neo4j-simple

[![Build Status](https://travis-ci.org/sebinsua/neo4j-simple.png)](https://travis-ci.org/sebinsua/neo4j-simple) [![npm version](https://badge.fury.io/js/neo4j-simple.svg)](https://npmjs.org/package/neo4j-simple)

**This is DEPRECATED.**

Simple [Neo4j](http://neo4j.com/) bindings for Node.js.

The library provides nodes and relationships in [the form of promises](https://github.com/kevinbeaty/any-promise) and is implemented on top of [Cypher queries](http://neo4j.com/developer/cypher-query-language/). Additionally *optionally* you can restrict database access through usage of [Joi](https://github.com/hapijs/joi) validators.

## Example

```javascript
var db = require('neo4j-simple')("http://localhost:7474");

var Node = db.defineNode({
  label: ['Example'],
  schema: {
    'name': db.Joi.string().required()
  }
});

var basicExampleNode = new Node({
  name: "This is a very basic example"
});

basicExampleNode.save().then(function (results) {
  console.log(results);
});
```

If the `name` (as shown above) had not been supplied when generating an instance of the Node class, then on `save()` an error would have been thrown from the promise.

The [Joi](https://github.com/hapijs/joi) schema can be passed into `defineNode()` either as `options.schema` or more explicitly as `options.schemas.default`.

**IMPORTANT:** You must have an index on your nodes `'id'` properties (configurable using the `idName` option) in order for the library to work correctly. If you do not, you will receive the error `Relationship was not found` on trying to `save()` a relationship. [Read about schema and legacy indexing here](http://neo4j.com/docs/stable/rest-api-schema-indexes.html).

A more involved example, might look like this:

```javascript
var Promise = require('bluebird');

// By default the idName passed into the library is 'id'.
// This means that you should have an `auto_index` on that
// field for both nodes and relationships.
var db = require('neo4j-simple')("http://localhost:7474", {
  idName: 'id'
});

// Multiple schemas can be passed in like so.
var Node = db.defineNode({
  label: ['Example'],
  schemas: {
    'default': {
      'id': db.Joi.string().optional()
    },
    'saveWithName': {
      'id': db.Joi.string().optional(),
      'name': db.Joi.string().required()
    }
});

var Relationship = db.defineRelationship({
  type: 'LOVE',
  schema: {
    'description': db.Joi.string()
  }
});

var example1 = new Node({
  id: "some-id-goes-here-1",
  name: "Example 1"
});
var example2 = new Node({
  id: "some-id-goes-here-2",
  name: "Example 2"
});
var example3 = new Node({
  id: "some-id-goes-here-3",
  name: "Example 3"
});

var exampleRelationship = new Relationship({
  description: "It's true",
}, [example1.id, example2.id], db.DIRECTION.RIGHT);

// If you explicitly pass in an operation then it will be used on validate for
// the schema lookup. By default the validator will check for a default schema
// which if empty will validate successfully.
// Additionally it will try to intelligently select a schema depending on the
// operation that is executing. For example: create, replace, update.
Promise.all([
  example1.save( { operation: 'saveWithName' } ),
  example2.save(),
  example3.save()
]).then(function (response) {
  return exampleRelationship.save();
});
```

# API

All of the methods that interact with the database return a promise.

## `defineNode(nodeDefinition)`

```javascript
{
  'label': [''], // Optional. Takes either a single string or array of labels.
  'schema': {}, // Optional. A default schema.
  'schemas': {}, // Optional. A key-value object of schemas can be passed in.
}
```

### `new Node(data, id)`

If an id is specified as the second argument then the node represents an update or replace operation. If this is not the case then the node represents a create operation and the id should either be found in `data.id` or a uuid will be automatically generated on `save()`.

#### `save(options)`

```javascript
{
  'operation': 'replace', // Optional. Pass in when you require a different schema to be tested.
  'replace': false // Optional. Defaults to false.
}
```

#### `remove()`

## `defineRelationship(relationshipDefinition)`

```javascript
{
  'type': '', // Optional. Takes a single string.
  'schema': {}, // Optional. A default schema.
  'schemas': {}, // Optional. A key-value object of schemas can be passed in.
}
```

### `new Relationship(data, ids, direction)`

#### `save(options)`

```javascript
{
  'operation': 'replace', // Optional. Pass in when you require a different schema to be tested.
  'replace': false // Optional. Defaults to false.
}
```

#### `remove()`

## `query(...)`

This is an alias of Rainbird's `query()` but will return a promise. See also `begin()`, `commit()`, `rollback()`, etc.

Rainbird supports multiple queries and can return multiple result sets. In our case `then()` will receive all of these results, however we supply a set of helper methods against the promise that make it easy to parse the results for the simpler case of one query.

### `getResults(alias, ..., alias3)`

This method assumes that the query named a node as `'n'` however the function is variadic and you can pass in as many identifiers as you wish to.

*NOTE: When you pass in multiple aliases, then this means the object returned will be keyed with these aliases and contain each node against the alias it was found at.*

e.g.

```javascript
var db = require('neo4j-simple')("http://localhost:7474");

db.query('MATCH (n:Example) RETURN n LIMIT 100')
  .getResults()
  .then(function (results) {
  console.log(results);
  // --> [{ properties of n }, { properties of n }]
});
```

### `getRelationshipResults(alias, ..., alias3)`

This method assumes that the query named a relationship as `'r'` and the nodes that this was between were `'n'` and `'m'`.

### `getResult(alias, ..., alias3)`

This method assumes that the query named a node as `'n'` however the function is variadic and you can pass in as many identifiers as you wish to.

*NOTE: When you pass in multiple aliases, then this means the object returned will be keyed with these aliases and contain each node against the alias it was found at.*

e.g.

```javascript
var db = require('neo4j-simple')("http://localhost:7474");

db.query('MATCH (n:User)-->(p:Product) RETURN n, p')
  .getResult('n', 'p')
  .then(function (result) {
  console.log(result);
  // --> { n: { properties of n}, p: { properties of p } }
});
```

### `getRelationshipResult(alias, ..., alias3)`

This method assumes that the query named a relationship as `'r'` and the nodes that this was between were `'n'` and `'m'`.

### `getCount()`

This method assumes that the query named the count as `count(n)`.

## `getNodes(ids)`

This is a method that executes an explicit query for a specific array of ids.

# Support

I am using it in an internal project so it is in active development. I will respond to any [issues](https://github.com/sebinsua/neo4j-simple/issues) raised. There will likely be breaking changes happening, however new versions will be released following [semver conventions](http://semver.org/) and over time I hope to increase the unit testing that I have done and to break some of it up into smaller components.

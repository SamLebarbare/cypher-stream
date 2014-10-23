# cypher-stream [![Build Status](https://travis-ci.org/brian-gates/cypher-stream.png?branch=master)](https://travis-ci.org/brian-gates/cypher-stream) [![NPM version](https://badge.fury.io/js/cypher-stream.png)](http://badge.fury.io/js/cypher-stream) [![devDependency Status](https://david-dm.org/brian-gates/cypher-stream.png?theme=shields.io)](https://david-dm.org/brian-gates/cypher-stream.png#info=devDependencies)

Neo4j cypher queries as node object streams.

## Installation
```
npm install cypher-stream
```

## Basic usage

``` js
var cypher = require('cypher-stream')('http://localhost:7474');

cypher('match (user:User) return user')
  .on('data', function (result){
    console.log(result.user.first_name);
  })
  .on('end', function() {
    console.log('all done');
  })
;
```

## Handling errors
``` js
var cypher = require('cypher-stream')('http://localhost:7474');
var should = require('should');
it('handles errors', function (done) {
  var errored = false;
  cypher('invalid query')
    .on('error', function (error) {
      errored = true;
      String(error).should.equal('Error: Query failure: Invalid input \'i\': expected SingleStatement (line 1, column 1)\n"invalid query"\n ^');
      error.neo4j.exception.should.equal('SyntaxException');
      error.neo4j.stacktrace.should.be.an.array;
      error.neo4j.statusCode.should.equal(400);
    })
    .on('end', function() {
      errored.should.be.true;
      done();
    })
    .resume() // need to manually start it since we have no on('data')
  ;
});

```

## Transactions


Transactions are duplex streams that allow you to write query statements then commit or roll back the written queries.

Transactions have three methods: `write`, `commit`, and `rollback`, which add queries and commit or rollback the queue respectively.

### Creating a transaction

``` js
var transaction = cypher.transaction(options)
```

### Adding queries to a transaction

``` js
transaction.write(query_statement);
```

A `query_statement` can either be a string or a query statement object.  A query statement object consists of a `statement` property and an optional `parameters` property.  Additionally, you can pass an array of either.

The following are all valid options:

``` js
var transaction = cypher.transaction();
transaction.write('match (n:User) return n');
transaction.write({ statement: 'match (n:User) return n' });
transaction.write({
  statement  : 'match (n:User) where n.first_name = {first_name} return n',
  parameters : { first_name: "Bob" }
});
transaction.write([
  {
    statement  : 'match (n:User) where n.first_name = {first_name} return n',
    parameters : { first_name: "Bob" }
  },
  'match (n:User) where n.first_name = {first_name} return n'
]);
```

### Committing or rolling back


``` js
transaction.commit();
transaction.rollback();
```

Alternatively, a query statement may also contain a `commit` or `rollback` property instead of calling `commit()` or `rollback()` directly.

``` js
transaction.write({ statement: 'match (n:User) return n', commit: true });
transaction.write({
  statement  : 'match (n:User) where n.first_name = {first_name} return n',
  parameters : { first_name: "Bob" },
  commit     : true
});

```

### Query Batching

Transactions automatically batch queries for significant performance gains.  Try the following:

``` js
    var queriesToRun = 10000;
    var queriesWritten = 0;
    var transaction = cypher.transaction()
      .on('data', function (result) {
        console.log(result);
      })
    ;
    while (queriesWritten++ < queriesToRun) {
      transaction.write('match (n:Test) return n limit 1');
    }
    transaction.commit();
```

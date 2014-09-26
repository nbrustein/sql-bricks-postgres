# PostgreSQL dialect for SQLBricks

This is a lightweight, schemaless library helping you to generate statements for PostgreSQL.
It is based on [sql-bricks](https://github.com/CSNW/sql-bricks) and adds PostgreSQL specific things into it.


## Installation

```
npm install sql-bricks-postgres
```


## Usage

```javascript
// in node:
var sql = require('sql-bricks-postgres');
// in the browser:
var sql = PostgresBricks;

sql.select().from('user').where({name: 'Fred Flintstone'}).toParams();
// -> {text: 'SELECT * FROM "user" WHERE name = $1', values: ['Fred Flintstone']}
```

You can read about basic flavor of how this thing works in [sql-bricks documentation](http://csnw.github.io/sql-bricks). Here go PostgreSQL specifics.


### LIMIT and OFFSET

```js
sql.select().from('user').limit(10).offset(20).toString();
// -> 'SELECT * FROM "user" LIMIT 10 OFFSET 20'
```


### RETURNING

```js
sql.update('user', {name: 'John'}).where({id: 1}).returning('*').toString();
// -> 'UPDATE "user" SET name = 'John' WHERE id = 1 RETURNING *'

sql.delete('job').where({finished: true}).returning('id').toString();
// -> 'DELETE FROM job WHERE finished = TRUE RETURNING id'
```


### UPDATE ... FROM

```js
sql.update('setting', {value: sql('V.value')})
   .from('val as V').where({name: sql('V.name')}).toString();
// -> 'UPDATE setting SET value = V.value FROM val as V WHERE name = V.name'
```


### DELETE ... USING

```js
sql.delete('user').using('address').where('user.addr_fk', sql('address.pk')).toString();
// -> 'DELETE FROM "user" USING address WHERE "user".addr_fk = address.pk');
```


### FROM VALUES

`VALUES` statement is a handy way to provide data with a query. It is most known in a context of `INSERT`, but could be used for other things like altering selects and doing mass updates:

```js
var data = [{name: 'a', value: 1}, {name: 'b', value: 2}];
sql.select().from(sql.values(data)).toString();
// -> "SELECT * FROM (VALUES ('a', 1), ('b', 2))");

sql.update('setting s', {value: sql('v.value')})
   .from(sql.values({name: 'a', value: 1}).as('v').columns())
   .where('s.name', sql('v.name')}).toString()
// -> "UPDATE setting s SET value = v.value
//     FROM (VALUES ('a', 1)) v (name, value) WHERE s.name = v.name"
```

Sometimes you need types on values columns for query to work. You can use `.types()` method to provide them:

```js
var data = {i: 1, f: 1.5, b: true, s: 'hi'};
insert('domain', _.keys(data))
    .select().from(sql.values(data).as('v').columns().types())
    .where(sql.not(sql.exists(
        select('1').from('domain d')
        .where({'d.job_id': sql('v.job_id'), 'd.domain': sql('v.domain')}))))
// INSERT INTO domain (i, f, b, s)
// SELECT * FROM (VALUES ($5::int, $6::float, $7::bool, $8)) v (i, f, b, s)
// WHERE NOT EXISTS
//    (SELECT 1 FROM domain d WHERE d.job_id = v.job_id AND d.domain = v.domain)
```

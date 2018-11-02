---
layout: documentation
title: Materializations
sub_headers: ["Tables", "Incremental tables"]
---

# Materializations

A materialization defines a table, or view that will be created in your data warehouse.

To define a new materialization, create a `.sql` file in the `models` directory. The name of the file will be the name of the table created in your data warehouse.

```js
// myfirstmodel.sql
select 1 as test
```

Will create a `view` called `myfirstmodel.sql` in the default dataform schema defined in the [`dataform.json`](/configuration/#dataform.json) file.

There are several configuration options that can be applied to a materialization. These can be applied by calling the appropriate method within a `${}` block, or can be provided all as one using the [options syntax](#Options syntax).

## Tables

By default, materializations are created as views in your warehouse. To create a copy of the query result as table, you can use the `type` method to change the materialization type to `"table"`.

```js
${type("table")}
--
select 1 as test
```

## Incremental tables

Incremental tables allow you build a table incrementally, by only inserting data that has not yet been processed.

In order to define an incremental table we must set the `type` of the query to `"incremental"`, and provide a where clause.

For example, if we have a timestamp field in our source table called `ts`, then we provide a where statement that means we only processes rows that are newer than the latest value `ts` in our output table. The `where` function should be used inline as part of a query.

```js
${type("incremental")}
--
select a, b from sourcetable
  ${where(`ts > (select max(ts) from ${self()}`)}
```

Note: to use the `${self()}` syntax within the call to `where()`, you must provide a string in back-tick's \`\`, in order to use JavaScript's template string syntax.

Incremental tables automatically produce the necessary `create table` and `insert` statements.

For the above example, if the table does not exist the the following statement will be run:

```js
create or replace table dataform.incrementalexample as
  select a, b
  from sourcetable
```

Subsequent runs will then run the following statement:

```js
insert into dataform.incrementalexample (a, b)
  select a, b
  from sourcetable
  where ts > (select max(ts) from dataform.incrementaltable)
```

It's important to note that incremental tables MUST specifically list selected fields, so that the insert statement can be automatically generated.

The following would not work:
```js
${type("incremental")}
--
select * from sourcetable
```

## Pre hooks

You can execute one or more statements before a table is materialized using the [`pre()`](/built-in-functions#pre) built-in:

```js
${pre([
  "run this before",
  "then run this before"
])}
--
select 1 as test
```

## Post hooks

You can execute one or more statements after a table is materialized using the [`post()`](/built-in-functions#post) built-in:

```js
${post([
  "run this after",
  "then run this after"
])}
--
select 1 as test
```

## Assertions

[Assertions](/assertions) can be easily added to a query without having to define them in a seperate file.

```js
select 1 as test
--
${assert([
  `select test from ${self()} where test > 1`,
])}
```

The above statement will check that the output of these query contains no rows where the `test` value is greater than 1.
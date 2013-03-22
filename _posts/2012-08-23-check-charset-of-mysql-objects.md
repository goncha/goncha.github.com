---
layout: post
title: "Check Charset of MySQL Objects"
tags: mysql
---

Check charset of databases and columns in MySQL.

Database:

    select schema_name, default_character_set_name from information_schema.schemata;

Column:

    select 
      t.table_schema,
      t.table_name,
      c.column_name,
      c.character_set_name
    from 
      information_schema.tables as t join information_schema.columns as c using (table_schema, table_name)
    where
      table_schema='YOUR_TABLE_SCHEMA' and c.character_set_name is not null;
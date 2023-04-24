---
title: Python Dataclasses
feed: show
date: 2023-03-07
tag: python
---

# Introduction

Python `dataclasses` are the bomb.  When modeling some data next time in Python, try using the
`dataclasses` library, which is available from Python 3.7.  Not only are they fast, they come with
a bunch of features that make them incredibly powerful.  They aren't
perfect, and we'll go over some of their shortcomings below, but you'll find them to be an
invaluable part of your Python toolkit.

# Basics

Let's start by modeling some data.

```python
from dataclasses import dataclass

@dataclass
class Car:
  make: str
  model: str
  year: int
  price: float
  num_wheels: int = 4
  metadata: dict | None = None
```

Dataclasses automatically generate an `__init__` method for you, with arguments in the same order
of the class attributes you define. You can set defalults for these attirbutes as shown for
`num_wheels` above.

```python
c = Car('Honda', 'CRV', 2014, 32_000.00)
```

Note that the types of the arguments are not enforced out of the box, meaning we can do something
like this without getting any error from Python.

```python
c = Car('Honda', 'CRV', '32000', 2014)
```

Dataclasses come with a special method hook called `__post_init__`, which is run after the
`__init__` method.  You can use it to enforce validations on your data, modify attributes, or add new
attributes based on existing ones.

```python
import json
from datetime import datetime
from dataclasses import field, fields


@dataclass
class Car:
  ...
  
  age: int = field(init=False)
  metadata: dict = None
  
  def __post_init__(self):
    # calculate new attribute based on existing attributes
    self.age = datetime.now().year - self.year
    
    # enforce types 
    self.validate_types()
    
    # serialize some fields to JSON
    self.metadata = json.dumps(self.metadata)
```

You can set the default value for each field of a dataclass, or otherwise customize it's behavior
using `field`. Be sure to check out [all of the
options](https://docs.python.org/3/library/dataclasses.html#dataclasses.field).

Similarly, you can use the `fields` method to inspect the fields of a dataclass.

Additionally, the `__dict__` method of a dataclass instance gives you a dictionary representation
of the data.

```python
  def validate_types(self):
    for f in fields(self):
      val = self.__dict__[f.name]
      if not isinstance(val, f.type):
        raise TypeError(f"field '{f.name}' should be of type {f.type} instead of {type(val)}")
```

# Sample Usage: A Minimal ORM

Let's use `dataclasses` to make a minimal ORM.

When starting a new project, typically we need some way to model data, and store and retrieve it
from a database. Instead of reaching for a heavy-handed library and bloating your project with
dependencies early on, try starting with some simple functions that cover most of the functionality
that you need. Your needs may outgrow it in the future, sure, but along the way, you might learn
about what you actually need, helping you make a better decision later.

## Intializing Tables

To start, let's generate some SQL to create a table for the class above.

```python
CREATE_TABLE_SQL = '''CREATE TABLE IF NOT EXISTS {} (\n  {}\n)'''

TYPE_SQL_MAP = {
    'int': 'INTEGER',
    'float': 'REAL',
    'str': 'TEXT',
    }

def create_table_sql(cls) -> str:
  fields_data: list[str] = []

  for f in fields(cls):
    type_name = f.type.__name__

    # default type to text, for example if a field is an anum
    sql_type = TYPE_SQL_MAP.get(type_name, 'TEXT')
    
    fields_data.append(f'{f.name} {sql_type}')
    
  return CREATE_TABLE_SQL.format(
      cls.__name__.lower(),
      ',\n  '.join(fields_data))
```

We can include a dictionary of `metadata` for each field in the `field` method, such as information
that we want to include in the table defintion. By default, the generated `__init__` method has all
positional arguments. We can set `kw_only` in a `field` to make it a keyword argument, or apply it
to all fields by setting it in the dataclass decorator.

```python
from dataclasses import field
...

@dataclass(kw_only=True)
class Car:
  id: int = field(metadata={'PRIMARY KEY': True}, init=False, default=None)
  ...
  
  date: str = field(metadata={'DEFAULT': 'CURRENT_DATE'}, init=False, default=None)
```

Here we are processing the `metadata` dictionary to process the extra metadata and generate the
proper SQL for each column.

```python
def create_table_sql(cls) -> str:
  ...
  for f in fields(cls):
    ...
    # by default set NOT NULL for all fields
    # merge the two dicts
    metadata: dict[str, bool | str] = dict({'NOT NULL': True}, **f.metadata)
    extra_args = [
        f'{k}{"" if v == True else " "+str(v)}'
        for k, v in metadata.items()
        if v]

    fields_data.append(f'{f.name} {sql_type}{" ".join([""] + extra_args)}')
  ...
```

This would generate the following SQL.

```sql
CREATE TABLE IF NOT EXISTS car (
  id INTEGER NOT NULL PRIMARY KEY,
  make TEXT NOT NULL,
  model TEXT NOT NULL,
  year INTEGER NOT NULL,
  price REAL NOT NULL,
  num_wheels INTEGER NOT NULL,
  metadata TEXT NOT NULL,
  date TEXT NOT NULL DEFAULT CURRENT_DATE
)
```

## Inserting Data

We can similarly generate SQL to insert a dataclass instance into the database.

```python
INSERT_SQL = '''INSERT INTO {} ({}) VALUES ({})'''

def insert_sql(ins) -> tuple[str, list]:
  # look up values in the dict representation
  column_names = [f.name for f in fields(ins) if f.init]
  values = [ins.__dict__[name] for name in column_names]
  # automatically generate number of '?'
  sql = INSERT_SQL.format(
      ins.__class__.__name__.lower(),
      ', '.join(column_names),
      ', '.join(['?']*len(values)))

  return sql, values
```

## Retrieving Data

TODO

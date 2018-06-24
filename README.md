# CosmicQL

HTTP APIs query language.

![GitHub tag](https://img.shields.io/github/tag/stellar-labs/cosmicql.svg)

## Summary

- [Introduction](#introduction)
- [Why choosing CosmicQL over GraphQL or REST?](why-choosing-cosmicql-over-graphQL-or-rest)
- [Fetching data](#fetching-data)
- [Reserved columns names](#reserved-columns-names)
- [Reserved tables names](#reserved-tables-names)
- [Reserved queries names](#reserved-queries-names)
- [Reserved fields modifiers names](#reserved-fields-modifiers-names)

## Introduction

One of the biggest critic over REST is its low performance because it has no built-in mecanism for filtering columns, and hacks like `?filter=col1,col2` came quickly to their limit when reducing the columns of the relationships.
REST suffers also from his lack of maintenance regarding datatabse change (versioning our endpoints seems to be the norme, but it still leads to frictions for the front-end developer when it comes to update manage different versions living in the same view, and it is not a standard behavior).

GraphQL solves this problem with an innovative way of querying the database, leading in a single endpoint and many advantages, that make it a serious challenger when it comes to performance and data saving.

However, after trying to implement GraphQL for PHP, I struggled in the parsing step. As the GraphQL message is a string, it requires to be first interpreted by a parsing engine, which become complex.
GraphQL can also slow down the development when it comes to complex query like relationships (which introduce new concepts like Types, and force to remap the entiere table).

CosmicQL is a thin layer between the database and the view, and rely on a very few concepts to make the implementation the easiest possible, without compormising the time and data saved.

## Why choosing CosmicQL over GraphQL or REST?

The advantage of CosmicQL is in the fact it requires a very poor learning curve, and very less resource to implement it, both on client and server side.

That's it! Really, if you do not mind this kind of concern, GraphQL is the way to go, REST has unfortunatly made his time (in my point of view).

## Fetching the data

- [Simple example](#simple-example)
- [Requesting relations](#requesting-relations)
- [Fields modifiers](#fields-modifiers)

### Simple example

Let us say we want to fetch all the existing tasks.

```json
{
    "task": {

    }
}
```

Ommiting everything, all the rows of this table will be returned, with for each rows all the columns.

```json
{
    "task": [
        { "id": 1, "name": "Finish this standard", "state": 1, "created_at": "2018-01-01T19:38:00+02:00" },
        { "id": 2, "name": "Dive more into PhalconPHP", "state": 2, "created_at": "2018-03-04T11:21:38+02:00" },
        { "id": 3, "name": "Contribute to StackOverflow", "state": 1, "created_at": "2018-06-23T14:41:07+02:00" }
    ]
}
```

We do not use the `state` for the moment, it will be useful only when we will talk of relations. Let us remove it for the moment.

```json
{
    "task": {
        "exclude": ["state"]
    }
}
```

Now we have the data that matter for us.

```json
{
    "task": [
        { "id": 1, "name": "Finish this standard", "created_at": "2018-01-01T19:38:00+02:00" },
        { "id": 2, "name": "Dive more into PhalconPHP", "created_at": "2018-03-04T11:21:38+02:00" },
        { "id": 3, "name": "Contribute to StackOverflow", "created_at": "2018-06-23T14:41:07+02:00" }
    ]
}
```

We only want to show the task that are pending (state: 1) and not the task in progress (state: 2). Let us fix this.

```json
{
    "task": {
        "exclude": ["state"],
        "where": {
            "state": 1
        }
    }
}
```

Now we have something closer to what we want.

```json
{
    "task": [
        { "id": 1, "name": "Finish this standard", "created_at": "2018-01-01T19:38:00+02:00" },
        { "id": 3, "name": "Contribute to StackOverflow", "created_at": "2018-06-23T14:41:07+02:00" }
    ]
}
```

Ok now let us say we want to filter to the month of june only. Might be a datepicker in the client side for instance.

```json
{
    "task": {
        "exclude": ["state"],
        "where": {
            "state": 1,
            "created_at": ["between", "2018-01-06", "2018-30-06"]
        }
    }
}
```

Now we have what we was looking for now.

```json
{
    "task": [
        { "id": 3, "name": "Contribute to StackOverflow", "created_at": "2018-06-23T14:41:07+02:00" }
    ]
}
```

Ok now we want the **first** task that have been created in june and that we should dispatch to our collegues.

```json
{
    "task": {
        "exclude": ["state"],
        "where": {
            "state": 1,
            "created_at": ["between", "2018-01-06", "2018-30-06"]
        },
        "limit": 1
    }
}
```

Finally, we have what we wanted.

```json
{
    "task": {
        "id": 3, 
        "name": "Contribute to StackOverflow", 
        "created_at": "2018-06-23T14:41:07+02:00" 
    }
}
```

Have complicated queries? Let the server take care of it.

```json
{
    "@taskCount": {

    }
}
```

We will get this response.

```json
{
    "taskCount": {
        "count": 3
    }
}
```

With the following server setup (PHP in this example).

```php
use StellarLabs\ComsmicQl;

CosmicQl::connection('default', [
    'host' => 'localhost',
    'database' => 'task',
    'username' => 'root',
    'password' => '',
    'charset' => 'utf8',
    'driver' => 'mysql'
]);

// ...

CosmisQl::raw('taskCount', "SELECT COUNT(1) AS 'count' FROM task");

CosmicQl::listen();
```

Complicated queries need parameters?

```json
{
    "@taskCount": {
        "where": {
            "state": 1
        }
    }
}
```

Which will produce the following.

```json
{
    "taskCount": {
        "count": 2
    }
}
```

Assuming we have the following server side raw query (this example feauring PHP).

```php
// ...

CosmicQl::raw('taskCount', "SELECT COUNT(1) AS 'count' FROM task WHERE state = :state");

// ...
```

Need to make several queries?

```json
{
    "taks": {
        "include": ["id", "name"]
    },
    "state": {
        "exclude": ["slug"]
    },
    "@taskCount": {

    }
}
```

No problem.

```json
{
    "task": [
        { "id": 1, "name": "Finish this standard" },
        { "id": 2, "name": "Dive more into PhalconPHP" },
        { "id": 3, "name": "Contribute to StackOverflow" }
    ],
    "state": [
        { "id": 1, "name": "pending" },
        { "id": 2, "name": "in progress" },
        { "id": 3, "name": "completed" }
    ],
    "taskCount": {
        "count": 3
    }
}
```

One filter or another?

```json
{
    "task": {
        "include": ["id", "name"],
        "where": {
            "state": 1,
            "or": {
                "state": 2
            }
        }
    }
}
```

Very well.

```json
{
    "task": [
        { "id": 1, "name": "Finish this standard", "state": 1, "created_at": "2018-01-01T19:38:00+02:00" },
        { "id": 2, "name": "Dive more into PhalconPHP", "state": 2, "created_at": "2018-03-04T11:21:38+02:00" },
        { "id": 3, "name": "Contribute to StackOverflow", "state": 1, "created_at": "2018-06-23T14:41:07+02:00" }
    ]
}
```

Agregating your data.

```json
{
    "state": {
        "include": [{"count": "id", "as": "taskCount"}, "name"],
        "groupBy": ["id"]
    }
}
```

This will report the following result.

```json
{
    "state": [
        { "taskCount": 2, "name": "pending" },
        { "taskCount": 1, "name": "in progress" },
        { "taskCount": 0, "name": "completed" }
    ]
}
```

Data can be filtered as well.

```json
{
    "tasl": {
        "exclude": ["state"],
        "order": {
            "id": "desc"
        }
    }
}
```

Reverse order task list.

```json
{
    "task": [
        { "id": 3, "name": "Contribute to StackOverflow", "created_at": "2018-06-23T14:41:07+02:00" },
        { "id": 2, "name": "Dive more into PhalconPHP", "created_at": "2018-03-04T11:21:38+02:00" },
        { "id": 1, "name": "Finish this standard", "created_at": "2018-01-01T19:38:00+02:00" }
    ]
}
```

### Requesting relations

If you saw the previous example, the state field was not very explicit... That is why we removed it. 

We will incorporate it back to our result list, and to make it more clear we will call the relation of the task: the state table.

```json
{
    "task": {
        "exclude": ["state"],
        "with": {
            "state": {
                "include": ["name"]
            }
        }
    }
}
```

This will produce the following result.

```json
{
    "task": [
        { 
            "id": 1, 
            "name": "Finish this standard", 
            "created_at": "2018-01-01T19:38:00+02:00",
            "state": {
                "name": "pending"
            }
        },
        { 
            "id": 2, 
            "name": "Dive more into PhalconPHP", 
            "created_at": "2018-03-04T11:21:38+02:00",
            "state": {
                "name": "in progress"
            }
        },
        { 
            "id": 3, 
            "name": "Contribute to StackOverflow", 
            "created_at": "2018-06-23T14:41:07+02:00",
            "state": {
                "name": "pending"
            }
        }
    ]
}
```

Let us abstract ourselves to only the pending tasks. We should filter the relation for it.

```json
{
    "task": {
        "exclude": ["state"],
        "with": {
            "state": {
                "include": ["name"],
                "where": {
                    "id": 1
                }
            }
        }
    }
}
```

This will trim the result a little bit.

```json
{
    "task": [
        { 
            "id": 1, 
            "name": "Finish this standard", 
            "created_at": "2018-01-01T19:38:00+02:00",
            "state": {
                "name": "pending"
            }
        },
        { 
            "id": 3, 
            "name": "Contribute to StackOverflow", 
            "created_at": "2018-06-23T14:41:07+02:00",
            "state": {
                "name": "pending"
            }
        }
    ]
}
```

Sometimes the relation is not mandatory.

```json
{
    "task": {
        "exclude": ["state"],
        "maybeWith": {
            "state": {
                "include": ["name"]
            }
        }
    }
}
```

This is what we get.

```json
{
    "task": [
        { 
            "id": 1, 
            "name": "Finish this standard", 
            "created_at": "2018-01-01T19:38:00+02:00",
            "state": {
                "name": "pending"
            }
        },
        { 
            "id": 2, 
            "name": "Dive more into PhalconPHP", 
            "created_at": "2018-03-04T11:21:38+02:00",
            "state": {
                "name": "in progress"
            }
        },
        { 
            "id": 3, 
            "name": "Contribute to StackOverflow", 
            "created_at": "2018-06-23T14:41:07+02:00",
            "state": {
                "name": "pending"
            }
        },
        {
            "id": 4,
            "name": "Draft (to finish...)",
            "created_at": "2018-23-06T20:23:01+02:00",
            "state": null
        }
    ]
}
```

No relation setup in the server-side? Fear no more.

```json
{
    "task": {
        "exclude": ["state"],
        "with": {
            "state": {
                "on": {
                    "id": ["task", "state_id"]
                },
                "include": ["name"]
            }
        }
    }
}
```

Still the same result.

```json
{
    "task": [
        { 
            "id": 1, 
            "name": "Finish this standard", 
            "created_at": "2018-01-01T19:38:00+02:00",
            "state": {
                "name": "pending"
            }
        },
        { 
            "id": 2, 
            "name": "Dive more into PhalconPHP", 
            "created_at": "2018-03-04T11:21:38+02:00",
            "state": {
                "name": "in progress"
            }
        },
        { 
            "id": 3, 
            "name": "Contribute to StackOverflow", 
            "created_at": "2018-06-23T14:41:07+02:00",
            "state": {
                "name": "pending"
            }
        }
    ]
}
```

### Fields modifiers

Modifier let you proceed basic operation to let you free of this job.

_Certain modifications that are not supported by the database operate on every rows after the query have been fetched. In this case it can slow the queries if you request a big amount of rows. You should take this into account._

- [Modifiers usage](#modifiers-usage)
- [Modifiers list](#modifiers-list)

#### Modifiers usage

Getting the dates of each taks as timestamps:

```json
{
    "task": {
        "include": ["name", {"timestamp": "created_at"}]
    }
}
```

This will produce the following result.

```json
{
    "task": [
        { "name": "Finish this standard", "created_at": "1514835480" },
        { "name": "Dive more into PhalconPHP", "created_at": "1520162498" },
        { "name": "Contribute to StackOverflow", "created_at": "1529764867" }
    ]
}
```

Another example: uppercases.

```json
{
    "task": {
        "include": [{"uppercase": "name"}, "created_at"]
    } 
}
```

With the follwing result.

```json
{
    "task": [
        { "id": 1, "name": "FINISH THIS STANDARD", "created_at": "2018-01-01T19:38:00+02:00" },
        { "id": 2, "name": "DIVE MORE INTO PHALCONPHP", "created_at": "2018-03-04T11:21:38+02:00" },
        { "id": 3, "name": "CONTRIBUTE TO STACKOVERFLOW", "created_at": "2018-06-23T14:41:07+02:00" }
    ]
}
```

Possibility to add custom mdifiers on the server side (in PHP for instance):

```php
CosmicQl::modifier('timestamp', funtion($input) {
    return (new DateTime($input))->getTimestamp();
});
```

#### Modfiers list

| driver | modifier  | native support | example (input)           | example (output) |
|--------|-----------|----------------|---------------------------|------------------|
| mysql  | uppercase | ✓              | foo                       | FOO              |
| mysql  | timestamp | ✘              | 2018-01-01T15:59:38+02:00 | 1514822378       |

## Reserved columns names

- `or`

## Reserved tables names

- `debug`

## Reserved raw queries names

- `debug`

## Reserved fields modifers names

- `avg`
- `sum`
- `max`
- `min`
- `count`
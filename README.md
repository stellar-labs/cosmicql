# CosmicQL

HTTP APIs query language.

![GitHub tag](https://img.shields.io/github/tag/stellar-labs/cosmicql.svg)

## Summary

- [Introduction](#introduction)
- [Why choosing CosmicQL over GraphQL or REST?](why-choosing-cosmicql-over-graphQL-or-rest)
- [Fetching data](#fetching-data)
- [Creating new data](#creating-new-data)
- [Updating data](#updating-data)
- [Deleting data](#deleting-data)
- [Restoring data](#restoring-data)
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

The other advantage over GraphQL is it is a lower level language for sending query to the database, meaning that you can do a maximum of task before having to create a raw query.

Finally, it is performance oriented, and not mutation-oriented. This means instead of tracking the modifications and being able to revert them, thing that CosmicQL cannot do, you can leverage some heavy tasks like batch insert and updates, or multiple queries in one single **server** request (but as many database requests as needed of course).

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
        { "id": 1, "name": "Finish this standard", "state_id": 1, "created_at": "2018-01-01T19:38:00+02:00" },
        { "id": 2, "name": "Dive more into PhalconPHP", "state_id": 2, "created_at": "2018-03-04T11:21:38+02:00" },
        { "id": 3, "name": "Contribute to StackOverflow", "state_id": 1, "created_at": "2018-06-23T14:41:07+02:00" }
    ]
}
```

We do not use the `state` for the moment, it will be useful only when we will talk of relations. Let us remove it for the moment.

```json
{
    "task": {
        "exclude": ["state_id"]
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
        "exclude": ["state_id"],
        "where": {
            "state_id": 1
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
        "exclude": ["state_id"],
        "where": {
            "state_id": 1,
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
        "exclude": ["state_id"],
        "where": {
            "state_id": 1,
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

Complicated queries need parameters?

```json
{
    "@taskCount": {
        "where": {
            "state_id": 1
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
            "state_id": 1,
            "or": {
                "state_id": 2
            }
        }
    }
}
```

Very well.

```json
{
    "task": [
        { "id": 1, "name": "Finish this standard" },
        { "id": 2, "name": "Dive more into PhalconPHP" },
        { "id": 3, "name": "Contribute to StackOverflow" }
    ]
}
```

Note this would be the same as:

```json
{
    "task": {
        "include": ["id", "name"],
        "where": {
            "state_id": ["in", 1, 2]
        }
    }
}
```

With the same result.

```json
{
    "task": [
        { "id": 1, "name": "Finish this standard" },
        { "id": 2, "name": "Dive more into PhalconPHP" },
        { "id": 3, "name": "Contribute to StackOverflow" }
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
    "task": {
        "exclude": ["state_id"],
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
        "exclude": ["state_id"],
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
        "exclude": ["state_id"],
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
        "exclude": ["state_id"],
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
        "exclude": ["state_id"],
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
        { "name": "FINISH THIS STANDARD", "created_at": "2018-01-01T19:38:00+02:00" },
        { "name": "DIVE MORE INTO PHALCONPHP", "created_at": "2018-03-04T11:21:38+02:00" },
        { "name": "CONTRIBUTE TO STACKOVERFLOW", "created_at": "2018-06-23T14:41:07+02:00" }
    ]
}
```

#### Modfiers list

| modifier     | mysql native support | example (input)     | example (output) |
|--------------|----------------------|---------------------|------------------|
| uppercase    | ✓                    | foo bar             | FOO BAR          |
| lowercase    | ✓                    | FOO BAR             | foo bar          |
| trim         | ✓                    |  foo bar            | foo bar          |
| wordcase     |                      | foo bar             | Foo Bar          |
| encodeBase64 | ✓                    | foo                 | Zm9v             |
| decodeBase64 | ✓                    | Zm9v                | foo              |
| timestamp    | ✓                    | 2018-01-01 15:59:38 | 1514822378       |
| year         | ✓                    | 2018-01-01 15:59:38 | 2018             |
| month        | ✓                    | 2018-01-01 15:59:38 | 1                |
| day          | ✓                    | 2018-01-01 15:59:38 | 1                |
| hour         | ✓                    | 2018-01-01 15:59:38 | 15               |
| minute       | ✓                    | 2018-01-01 15:59:38 | 59               |
| second       | ✓                    | 2018-01-01 15:59:38 | 38               |
| time         | ✓                    | 2018-01-01 15:59:38 | 15:59:38         |

## Creating new data

We will stick to the task example. Let us create a new task:

```json
{
    "task": {
        "add": [
            { "name": "add new useful modifiers", "state_id": 1 }
        ]
    }
}
```

This will return the list of all the task id that have been created.

```json
{
    "task": [5]
}
```

If you need, you can add several rows.

```json
{
    "task": {
        "add": [
            { "name": "add new useful modifiers", "state_id": 1 },
            { "name": "buy a tempered glass for my phone", "state_id": 1 },
            { "name": "complete my leg day", "state_id": 1 }
        ]
    }
}
```

The list of ids will be the following.

```json
{
    "task": [5, 6, 7]
}
```

You can do several batch insert in one query.

```json
{
    "task": {
        "add": [
            { "name": "add new useful modifiers", "state_id": 1 },
            { "name": "buy a tempered glass for my phone", "state_id": 1 },
            { "name": "complete my leg day", "state_id": 1 }
        ]
    },
    "state": {
        "add": [
            { "name": "canceled", "slug": "canceled" }
        ]
    }
}
```

With the same output.

```json
{
    "task": [5, 6, 7],
    "state": [4]
}
```

Inserts can be mixed with others queries like selection.

```json
{
    "task": {
        "add": [
            { "name": "add new useful modifiers", "state_id": 1 }
        ]
    },
    "state": {
        "include": ["name"],
        "where": {
            "state_id": 1
        }
    }
}
```

With the following result.

```json
{
    "task": [5],
    "state": [
        { "name": "pending" }
    ]
}
```

### Updating data

To update a row, use its id, followed by the fixes.

```json
{
    "task": {
        "fix": [
            { "id": 1, "name": "Complete this standard", "state_id": 2 }
        ]
    }
}
```

This will return the ids of fixed tasks.

```json
{
    "task": [1]
}
```

CosmicQL supports batch updates as well.

```json
{
    "task": {
        "fix": [
            { "id": 1, "name": "Complete this standard", "state_id": 2 },
            { "id": 2, "name": "Dive into Micro PhalconPHP apps" }
        ]
    }
}
```

Always returning the ids afterwards.

```json
{
    "task": [1, 2]
}
```

... Alright, you might want to omit the `primaryKey` in case of conditionnal batch update. Got it guys...

```json
{
    "task": {
        "fix": {
            "name": "task completed"
        },
        "where": {
            "state_id": 1
        }
    }
}
```

Better?

```json
{
    "task": [1, 3]
}
```

But beware, for safety reasons, you can only batch update the entire table only with a trivial where clause like `true, true` to force your consent.

```json
{
    "task": {
        "fix": {
            "state_id": 1
        },
        "where": {
            "true": true
        }
    }
}
```

You can update a record, and if its it does not exist, automatically create it with the value provided.

_This will only work for sure with relational database that support `INSERT ... ON DUPLICATE KEY UPDATE ...` syntax._

```json
{
    "task": {
        "fixOrAdd": [
            { "id": 1, "name": "Complete this standard", "state_id": 2 }
        ]
    }
}
```

With the same result as usual.

```json
{
    "task": [1]
}
```

_You cannot use the `where` variant because it always expect a `primaryKey` to check if the row does not already exist._

## Deleting data

Deleting a data is not the removal of the data. It is just a convenient way to keep the data in your database, but hide it from your select queries. Think of it like a trash bin.

```json
{
    "task": {
        "trash": [1, 3]
    }
}
```

For convenience, the ids of the trashed rows are returned.

```json
{
    "task": [1, 3]
}
```

Of course, this supports trashing across several tables.

```json
{
    "task": {
        "trash": [1, 3]
    },
    "state": {
        "trash": [4, 5]
    }
}
```

With the same result.

```json
{
    "task": [1, 3],
    "state": [4, 5]
}
```

And of course you can mix the query types.

```json
{
    "task": {
        "include": ["id", "name"]
    },
    "state": {
        "trash": [4, 5]
    }
}
```

With the expected return value.

```json
{
    "task": [
        { "name": "Finish this standard" },
        { "name": "Dive more into PhalconPHP" },
        { "name": "Contribute to StackOverflow" },
        { "name": "add new useful modifiers" }
    ],
    "state": [4, 5]
}
```

## Restoring data

After trashing one or several rows, you can revert this.

```json
{
    "task": {
        "restore": [1, 3]
    }
}
```

It returns the ids of the restored rows.

```json
{
    "task": [1, 3]
}
```

Again, this supports muliple restoring like every actions.

```json
{
    "task": {
        "restore": [1, 3]
    },
    "state": {
        "restore": [4, 5]
    }
}
```

It also supports variant query actions... yes like every others actions how did you guessed ?!

```json
{
    "task": {
        "restore": [1, 3]
    },
    "state": {
        "include": ["id", "name"]
    }
}
```

With the regular results.

```json
{
    "task": [1, 3],
    "state": [
        { "name": "pending" },
        { "name": "in progress" },
        { "name": "completed" }
    ]
}
```

## Removing data

Removing a data will delete it, and you will not be able to restore it. This is a regular `DELETE ...` SQL statement for instance.

```json
{
    "task": {
        "remove": [2, 4]
    }
}
```

Passing the ids of the rows we want to delete, this will return the ids as a delete confirmation.

```json
{
    "task": [2, 4]
}
```

Obviously, you can mix the queries.

```json
{
    "task": {
        "trash": [2, 4]
    },
    "state": {
        "include": ["name"],
        "where": {
            "id": ["in", 2, 4]
        }
    }
}
```

With the expected result.

```json
{
    "task": [2, 4],
    "state": [
        { "name": "in progress" },
        { "name": "canceled" }
    ]
}
```

## Reserved columns names

- `or`
- `true`

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
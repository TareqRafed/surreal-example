## WHAT IS SURREALDB!?

SurrealDB is a multi-model database, its engine is powered by Rust, and it delivers a massive amount of features, it's fast, lightweight, schemaless, schemafull, relational, ACID, on-memory, in-disk, embedded and the feature list goes on.

This year Surreal became open-source and 2 weeks ago, SurrealDB v1.0.0beta7 was released.


## Why is it so cool?

The flexibility that Surreal offers in using different methodologies for storing and querying data and the ability to use it in-memory like Redis or on-disk like traditional SQL databases as well as using it as a filesystem like SQLite, combined with its lightweight and the ability to scale to a massive amount of data, makes it interesting and worth to have a closer look at it. 

## Setup & Installation

I will install SurrealDB on a docker container, through docker compose, you can also check other installation methods on the official [documentation](https://surrealdb.com/install)


With that, I will just configure a docker-compose file in a new folder

```yml
version: '3.8'
services:
  db:
    image: surrealdb/surrealdb:latest
    restart: always
    # You can remove the memory option if you want to run the DB on disk
    command: start --user root --pass root memory
    ports:
      - '8000:8000'
    volumes: 
      - db:/var/lib/surrealdb/data
volumes:
  db:
    driver: local

```

then just run it.

```bash
$ docker compose up db -d
``` 

using any REST client, send a get request to ***http://localhost:8000/sql*** and you should get a response like this:

```json
{
	"code": 405,
	"details": "Request content length too large",
	"description": "The requested HTTP method is not allowed for this resource. Refer to the documentation for allowed methods."
}
```


## Running Queries on DB

There are different ways to run queries, either by on the terminal using surreal cli, by using their client module, which we installed earlier, or by using traditional RESTful HTTP URL requests.

In this article, I will use a REST client.


## Headers

I'm going to use the Insomnia REST client, but there are also other options like [Thunder](https://marketplace.visualstudio.com/items?itemName=rangav.vscode-thunder-client), a vscode built-in REST client extension, or [Postman](https://www.postman.com/).

In the header of the request, **NS** and **DB** keys should be added, where those references are the namespace and database names the queries should run on.

Also, the request uses basic authentication, in our case, username and password should be both **root**.

There should be as well content-type header and it's used to get the desired type of response.



## CREATE

By default, the Database is schemaless, which will allow the creation of data without any restrictions. 

It has a similar syntax to SQL, and it's also not case sensitive, but I will stick to the convention which is to capitalize the keywords.

Surreal supports Arrays & Objects with unlimited nesting depth, So I will create a new **person** with the name **Joe** and with an Array of **skills**


```sql
// Random ID for this record
CREATE person CONTENT {
	name: 'Joe',
	skills: ['Python', 'JavaScript'],
};

// Specific ID for this one
CREATE person:myspecificID CONTENT {
	name: 'Joe',
	skills: ['Python', 'JavaScript'],
};

// another syntax
CREATE person SET name = 'Joe', skills = ['Python', 'JavaScript'];

```


And you should get

```json
[
	{
		"time": "128.4µs",
		"status": "OK",
		"result": [
			{
				"id": "person:z0tsj5jf5g98n4w3arbl",
				"name": "Joe",
				"skills": [
					"Python",
					"JavaScript"
				]
			}
		]
	}
]

```


## SELECT

Just like SQL, you can get data from a specific table like:

```sql
SELECT * FROM person;

SELECT name FROM person WHERE name == "Joe";

SELECT skills FROM person WHERE "JavaScript" INSIDE skills AND "Python" INSIDE skills;

```

Or send a GET request with the required headers to ***/key/person***

## UPDATE

To update data for a specific record

```sql 
UPDATE person:93rp63z54y9eri32bhdm SET name = "John"
```

Or to update data for all records.

Records that have Rust in their skills array, won't be updated.

```sql 
UPDATE person SET skills += ["Rust"]
```

## INSERT

Let's create multiple users, with different fields using SQL traditional INSERT statement

```sql
INSERT INTO person (name, age) VALUES ('Mike', '24'), ('John', '37');
```

## DELETE

You can clear a table by using the **DELETE** keyword like this

```sql
DELETE person
```

or remove a specific record

```sql
DELETE table:id
```

or remove all record that has specific data

```sql
DELETE * FROM person WHERE skills CONTAINS "JavaScript"
```

## Relations

You can easily link fields with other tables by using the table and the id record

### One-to-One
```sql

CREATE person:joe CONTENT {
	name: 'Joe',
	skills: ['Python', 'JavaScript'],
};

CREATE person:zoe CONTENT {
	name: 'Zoe',
	skills: ['Rust', 'Go'],
};

UPDATE person:joe SET manager = person:zoe;
```

In SurrealDB there is no **JOIN**, instead, you can access data with .

```sql
SELECT manager.name FROM person
```

Returns

```json
[
	{
		"time": "641.4µs",
		"status": "OK",
		"result": [
			{
				"manager": {
					"name": null
				}
            },
			{
				"manager": {
					"name": "Zoe"
				}
			},
			{
				"manager": {
					"name": null
				}
            }
		]
	}
]
```

### One-to-Many

For this, I'll create a new table called **article**, where each article has a title and description and add some records.

```sql
CREATE article:javascript CONTENT {
       title: "Cool thing with JS",
       description: "Yeah, it's cool"
};

CREATE article:python CONTENT {
       title: "Cool thing with Py",
       description : "extraordinary cool"
};

CREATE article:rust CONTENT {
       title: "Cool thing with Rust",
       description: "So cool"
};

CREATE article:go CONTENT {
       title: "Cool thing with Go",
       description : "Pretty cool"
};
```

Then Joe will get JS and Py articles, while Zoe will get Rust and Go articles

```sql
UPDATE person:joe SET articles = ['article:javascript', 'article:python'];
UPDATE article:javascript SET author = person:joe;
UPDATE article:python SET author = person:joe;

UPDATE person:zoe SET articles = ['article:rust', 'article:go'];
UPDATE article:rust SET author = person:zoe;
UPDATE article:go SET author = person:zoe;
```

Then we can retrieve Joe's article titles like this

```sql
SELECT articles.title FROM person:joe
```
Returns

```json
[
  {
    "articles": {
      "title": [
        "Cool thing with JS",
        "Cool thing with Py"
      ]
    }
  }
]
```

Or by

```sql
SELECT title FROM article WHERE author = person:joe
```

Returns

```json
[
  {
    "title": "Cool thing with JS"
  },
  {
    "title": "Cool thing with Py"
  }
]
```

And if needed we can get author names of Rust and Py articles like

```sql
SELECT author.name FROM article WHERE title CONTAINS "Rust" OR title CONTAINS "Py"
```

Returns

```json
[
  {
    "author": {
      "name": "Joe"
    }
  },
  {
    "author": {
      "name": "Zoe"
    }
  }
]
```

## Scripting queries

A crazy feature as well is you can write literal JavaScript in queries!

```sql
UPDATE person:joe SET rate = function() {
	return [1,2,3].map(v => v * 10);
};
```


## Schemafull table

The previous examples were schemaless, where there are no restrictions on fields or types, the cool thing about SurrealDB is that you can use schemafull tables as well

```sql
DEFINE TABLE animal SCHEMAFULL;
DEFINE FIELD name ON TABLE animal TYPE string;
DEFINE FIELD age ON TABLE animal TYPE int;

CREATE animal CONTENT {
	name: "dog",
	age: 4
};

// stores age as 0
CREATE animal CONTENT {
	name: "dog",
	age: "not an int"
};

```

## Is it going to replace Redis?

While yes, SurrealDB can work in memory, it's far away to be a replacement for Redis.
Redis data structures are designed for a few specific use cases to make it run as fast as possible.


## Conclusion

While SurrealDB seems to have a bright future and I believe it's going to compete well against SQLite, it's not designed to replace any of the major databases like Redis, MySQL, or PostgreSQL.

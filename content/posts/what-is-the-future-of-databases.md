---
title: "What is the future of databases?"
date: 2021-01-23T14:20:44+01:00
draft: false 
categories:
  - Databases
---

As I was sitting at work, coming up with a way to solve some synchronization issue between our relational database and a search index, it got me thinking... Is it possible to have a database that can handle this stuff for me, the way, for example, Postgres handles  the creation and maintenance of an index? Just one **CREATE INDEX** away from efficient and fast queries. What about a database that can serve every need: storing and reading data effectively, scaling smoothly as the workload increases, being able to run complex searches, do batch and stream processing for analytical purposes and work with every other access pattern we can imagine. 

Almost any organization that handles data uses a relational database in its stack. It makes sense, it's a battle-tested technology that stood the test of time. For a large percent of the applications, the need won't arise for a different kind of data system. Take Postgres for example: it's highly scalable, it has transactions with ACID properties, materialized views, supports partitioning and replication, joins, views, triggers, stored procedures, hot backups, all that kind of stuff. We can go
along way with it without having to worry about problems with the dataflow in our system. But of course, not all data models are suitable for Postgres, and even though it supports things like full-text searches and JSON columns, some times it is more suitable or easier even, to use different technology.

But even though relational databases have come a very long way, still they are not designed to handle every possible workload, so we resort to having to combine multiple systems to handle this broad variety: we use Redis for caching, Elasticsearch for searching, Kafka for stream processing, and many more similar tools and technologies, each of them serving a very narrow purpose in handling our different use cases.

But is there a way to unify the interface across these systems? 

These technologies are great at what they do, we just need something that nicely ties them together instead of putting that burden on our applications. Putting the application in this role is too much of a responsibility, storing and transporting the data from one place to another as well as handling all of the business logic is a sure way of making those systems permanently inconsistent with each other. 

Instead, if we can figure out a way of making these diverse components work nicely together, we will have a much more reliable solution. Kinda like the pipe in Unix, allowing the user to compose different workflows together out of simpler programs, by sending the output of one command to another, for further processing. 

Now I know there are tools today that handle some of these cases that I mentioned, and they are getting better each day, but they're far from simple to operate and maintain, and more importantly, they're not of declarative nature. What I would love to have is some abstraction akin to SQL, some declarative language that will hide away the complexities of storing and synchronizing the data across the different data systems, making it easier to combine the disparate technologies, while also providing escape hatches in case the need arises to access the underlying storage directly. 

Taking inspiration from the UNIX philosophy which emphasizes building complex systems out of simpler, extensible, and modular components, can we apply the same principles here, to our future data systems? 

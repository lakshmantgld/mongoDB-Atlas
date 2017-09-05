# MongoDB's Best Practices
This repo is a collection of mongoDB's best practices & features of Atlas. I was reading a bunch of articles, blogs & medium posts on mongoDB. So, I wanted to aggregate all the nuances and best practices into one place.  

## Table of Contents

- [Introduction to MongoDB](#Introduction to MongoDB)
- [What is Atlas?](#What is Atlas)
- [Data Modeling in mongoDB](#Data Modeling)
    - [Embedded R]
## Introduction to MongoDB:
At the time of writing, **MongoDB** is one of the **255** NoSQL databases. When compared to relational databases, [NoSQL](https://www.mongodb.com/nosql-explained) in general are more scalable and provide superior performance. Both SQL & NoSQL databases have their unique advantages and trade off. NoSQL databases are chose for their schema-less model and highly scalable system. According to [CAP](https://en.wikipedia.org/wiki/CAP_theorem) theorem, any distributed system can only satisfy two of these three Qualities namely, **C** onsistent, **A** vailability & **P** artition tolerance.

As we know that SQL database are [ACID](https://en.wikipedia.org/wiki/ACID) constraint, it qualifies being Consistent and highly Available. So it is difficult for the SQL system to scale horizontally. In case of NoSQL as they are eventually consistent, they are highly Available and Partition tolerance.

Coming back to mongoDB, it is a [document-oriented](https://www.mongodb.com/what-is-mongodb) database which stores record as documents.

## What is Atlas?
Having a single node mongoDB is quite easy. When we want to realize a multi-shared mongo cluster, managing the cluster is critical and a tiring job. [Atlas](https://www.mongodb.com/cloud/atlas) is mongoDB as a service, where the whole cluster is completely managed one.

## Data Modeling:
MongoDb is so called **schema-less** database and you can ask me why should we model such a database? Even though mongoDb is a schema-less database, it is necessary to model it, in order to achieve best performance. One thumb rule to consider in NoSQL, design schema based on your queries. Whereas in SQL you design based on the structure of the data.

In general, `Joins` are expensive operations in database. When you use a *Join(one-to-N)* in SQL, it will incur a time penalty. All we need to do is flatten the **one-to-N** relationships in mongoDB. Lets see how we can model the one-to-N relationships in mongoDB. In fact, Joins are not possible in mongoDB queries, but can be performed inside the application.

When designing a MongoDB schema, you need to start with a question that you would never consider when using SQL: *what is the cardinality of the relationship?* Put less formally: you need to characterize your “One-to-N” relationship with a bit more nuance:
![One-to-N](https://raw.githubusercontent.com/lakshmantgld/mongoDB-Atlas/master/readmeFiles/one-to-many.png)

is it “one-to-few”, “one-to-many”, or “one-to-squillions”? Depending on which one it is, you’d use a different format to model the relationship.

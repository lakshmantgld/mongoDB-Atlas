# MongoDB's Best Practices
This repo is a collection of mongoDB's best practices & features of Atlas. I was reading a bunch of articles, blogs & medium posts on mongoDB. So, I wanted to aggregate all the nuances and best practices into one place.  

## Table of Contents

- [Introduction to MongoDB](#Introduction-to-MongoDB)
- [What is Atlas?](#what-is-atlas)
- [Data Modeling in mongoDB](#data-modeling)
    - [One-to-Few](#One-to-Few)
    - [One-to-Many](#One-to-Many)
    - [One-to-Squillions](#One-to-Squillions)
- [Atlas Technical Architecture](#Atlas-Technical-Architecture)
- [Atlas Best Practices](#Atlas-Best Practices)
    - [Security](#Atlas-Security)
    - [Atlas Cluster Creation](#Atlas-Cluster-Creation)

## Introduction to MongoDB
At the time of writing, **MongoDB** is one of the **255** NoSQL databases. When compared to relational databases, [NoSQL](https://www.mongodb.com/nosql-explained) in general are more scalable and provide superior performance. Both SQL & NoSQL databases have their unique advantages and trade off. NoSQL databases are chose for their schema-less model and highly scalable system. According to [CAP](https://en.wikipedia.org/wiki/CAP_theorem) theorem, any distributed system can only satisfy two of these three Qualities namely, **C** onsistent, **A** vailability & **P** artition tolerance.

As we know that SQL database are [ACID](https://en.wikipedia.org/wiki/ACID) constraint, it qualifies being Consistent and highly Available. So it is difficult for the SQL system to scale horizontally. In case of NoSQL as they are eventually consistent, they are highly Available and Partition tolerance.

Coming back to mongoDB, it is a [document-oriented](https://www.mongodb.com/what-is-mongodb) database which stores record as documents.

## What is Atlas?
Having a single node mongoDB is quite easy. When we want to realize a multi-shared mongo cluster, managing the cluster is critical and a tiring job. [Atlas](https://www.mongodb.com/cloud/atlas) is mongoDB as a service, where the whole cluster is completely managed one.

## Data Modeling
MongoDb is so called **schema-less** database and you can ask me why should we model such a database? Even though mongoDb is a schema-less database, it is necessary to model it, in order to achieve best performance. One thumb rule to consider in NoSQL is to design the schema based on your queries, Whereas in SQL you design based on the structure of the data.

In general, `Joins` are expensive operations in database. When you use a *Join(one-to-N)* in SQL, it will incur a time penalty. In mongoDB, all we need to do is to flatten the **one-to-N** relationships. Lets see how we can model the one-to-N relationships in mongoDB. In fact, Joins are not possible in mongoDB queries, but can be performed inside the application.

When designing a MongoDB schema, you need to start with a question that you would never consider when using SQL: *what is the cardinality of the relationship?* Put less formally: you need to characterize your “One-to-N” relationship with a bit more nuance:
![One-to-N](https://raw.githubusercontent.com/lakshmantgld/mongoDB-Atlas/master/readmeFiles/one-to-many.png)

Is the cardinality `One-to-Few`, `One-to-Many`, or `One-to-Squillions`? Depending on which one it is, you would use a different format to model the relationship.

- ### One-to-Few

An example of **One-to-Few** might be the addresses for a person. This is a good use case for embedding – you would put the addresses in an array inside of your Person object. Make a note, the maximum size of a mongoDB document is `16 MB`.

```js
> db.person.findOne()
{
  name: 'Kate Monster',
  ssn: '123-456-7890',
  addresses : [
     { street: '123 Sesame St', city: 'Anytown', cc: 'USA' },
     { street: '123 Avenue Q', city: 'New York', cc: 'USA' }
  ]
}
```

##### Advantages:

- You don`t have to perform a separate query to get the embedded address details.
- Flattened the One-to-Many relationship. One query is enough to fetch the associated details.

##### Dis-Advantages:

- You have no way of accessing the embedded details as stand-alone entities.

As I previously said, you have to model your schema based on your queries. So in this case, assuming you don`t want to query addresses as stand-alone entities.

- ### One-to-Many

An example of **One-to-Many** might be parts for a product in a replacement parts ordering system. Each product may have up to several hundred replacement parts, but never more than a couple thousand or so. (All of those different-sized bolts, washers, and gaskets add up.) This is a good use case for referencing – you’d put the **ObjectIDs** of the parts in an array in product document. This type of referencing is called `child-referencing`, where you refer the child documents in the parent document.

Each Part would have its own document:

```js
> db.parts.findOne()
{
    _id : ObjectID('AAAA'),
    partno : '123-aff-456',
    name : '#4 grommet',
    qty: 94,
    cost: 0.94,
    price: 3.99
}
```

Each Product would have its own document, which would contain an array of ObjectID references to the Parts that make up that Product:

```js
> db.products.findOne()
{
    name : 'left-handed smoke shifter',
    manufacturer : 'Acme Corp',
    catalog_number: 1234,
    parts : [     // array of references to Part documents
        ObjectID('AAAA'),    // reference to the #4 grommet above
        ObjectID('F17C'),    // reference to a different Part
        ObjectID('D2AA'),
        // etc
    ]
}
```

You would then use an application-level join to retrieve the parts for a particular product:

```js
// Fetch the Product document identified by this catalog number
> product = db.products.findOne({catalog_number: 1234});
// Fetch all the Parts that are linked to this Product
> product_parts = db.parts.find({_id: { $in : product.parts } } ).toArray();
```

We should think NoSQL has **Not Only SQL**, meaning it can do additional operations in addition to the standard operations in SQL. We need to use the index feature of SQL in NoSQL, whenever the use case arises. In the above case, for efficient operation, you would need to have an index on `products.catalog_number`. Note that there will always be an index on `parts._id`, so that query will always be efficient.

##### Advantages:

- Each Part is a stand-alone document, so it is easy to search them and update them independently.
- You can still denormalize the schema by having frequently accessed part information inside the product document along with `parts._id`. This will reduce the Joins.

##### Dis-Advantages:

- The queries like show all the parts of the given product involves a **Join**, which has to be done in the application level.
- When you denormalize the schema, updates has to be done in two collections **Product** and **Part**.

- ### One-to-Squillions

An example of **One-to-Squillions** might be an event logging system that collects log messages for different machines. Any given host could generate enough messages to overflow the **16 MB** document size, even if all you stored in the array was the **ObjectID**. This is the classic use case for `parent-referencing` – you would have a document for the host, and then store the ObjectID of the host in the documents for the log messages.

```js
> db.hosts.findOne()
{
    _id : ObjectID('AAAB'),
    name : 'goofy.example.com',
    ipaddr : '127.66.66.66'
}

>db.logmsg.findOne()
{
    time : ISODate("2014-03-28T09:42:41.382Z"),
    message : 'cpu is on fire!',
    host: ObjectID('AAAB')       // Reference to the Host document
}
```

You would use a (slightly different) application-level join to find the most recent 5,000 messages for a host:

```js
// find the parent ‘host’ document
> host = db.hosts.findOne({ipaddr : '127.66.66.66'});  // assumes unique index
// find the most recent 5000 log message documents linked to that host
> last_5k_msg = db.logmsg.find({host: host._id}).sort({time : -1}).limit(5000).toArray()
```

As you can see, when you design the schema properly, you can scale mongoDB infinitely.

The above modeling is just the basics, there is still more to it like **Two-way** referncing and denormalization. I consider this [article](https://highlyscalable.wordpress.com/2012/03/01/nosql-data-modeling-techniques/) as bible to NoSQL modeling. Check out the brilliant article by mongoDB for the [best practices in mongoDB modeling](https://www.mongodb.com/blog/post/6-rules-of-thumb-for-mongodb-schema-design-part-1). The above tips were partly from this article. Since this repo is going to concentrate more on Best Practices with mongoDB Atlas, I am going to jump to that.

## Atlas Technical Architecture
The Technical architecture shows a Single shard mongoDB cluster deployed in AWS. This gives an higher level understanding of how it works.

![Atlas Technical Architecture](https://raw.githubusercontent.com/lakshmantgld/mongoDB-Atlas/master/readmeFiles/architecture.png)

you can deploy Atlas mongoDB cluster on any cloud provider of your choice. Since, our application is on AWS, we have deployed the atlas cluster on AWS. It securely communicates with our application through the **VPC peering**.

## Atlas Best Practices

### Security

- ##### VPC
The mongoDB cluster is deployed on mongoDB's VPC and you can connect to the cluster either through a publicly available URL or through VPC peering connection. But the best practice is to connect to your cluster using a peering connection, as the database will not be publicly exposed.

- ##### Database Cluster User & Password
With or Without a Peering connection, You still need a password to connect to the database. The best part with Atlas is you can give various privileges to the user. A user can be an `Atlas admin`, `Read & Write to a database` or `only Read access to database`. You can also assign fine-grained permissions like assigning above privileges to only a set of databases and collections. Once you create a user and the associated privileges, you get associated password which you can use to connect to your cluster.

**Use-Case:** The `only Read access to database` privilege will be proper fit for analytics, where the concerned user just wants to read the database, but have no permission to mutate it. Best practice is to **grant least privilege** required for the given user.

- ##### Atlas User
This user is different from the above cluster user. The **fundamental unit** in Atlas is the **mongoDB cluster**. Each project can contain multiple clusters. You can add users to your project with various privileges like `Admin` or `Read-Only`. As perviously said, the best practice is  **grant least privilege** required for the given user.

- ##### Encryption at Rest
Atlas clusters are deployed in the AWS`s EC2 instances. These instance use [EBS](https://aws.amazon.com/ebs/) for their storage. [KMS](https://aws.amazon.com/kms/) is another AWS service which uses Envelope Encryption techniques to protect your data in the AWS services. [EBS uses KMS for providing Encryption at rest](http://docs.aws.amazon.com/kms/latest/developerguide/services-ebs.html) which is leveraged by Atlas. The Encryption/Decryption is transparent, i.e the cryptographic process is not know to the user accessing the data from EBS. But **Beware**, this will introduce a latency in the reads/writes to the storage.

You can encrypt data at rest by using the above option. This option can be selected during the cluster creation process. The best practice is to select the option only. If you have any **SLA(service level agreement)** or **compliance** for data encrypt as this will introduce some latency, which affects the user experience.

- ##### Encryption in Flight
By default, SSL connection is used between your connection & the Atlas cluster. So, Encryption is made on fly, which neglects the data tampering. SSL connection is used even when you query your atlas backup.

### Atlas Cluster Creation

The cluster creation is surprisingly simple, you can deploy an entire Atlas cluster with few button clicks. I will list down the best practices in each step of creating the cluster.

- ##### Step 1
Once you are logged in to your **Project** -> **Clusters**, you will find the **Build a New Cluster** option in the upper right corner. Click that Button. you will see a long modal consisting steps for cluster creation process.

- ##### Cluster Name & MongoDB Version
In the modal, first option is **Cluster Name**, name you cluster meaningfully as you will not be able to update it later. Second option is **MongoDB Version**, the Atlas itself selects the latest version for you. But If you want to change to a particular mongoDB version, you can also do that.

- ##### Cloud Provider & Region
The third option is **Cloud Provider and Region**. Choose the cloud provider you wish and make sure you select a region similar to your application's region. This is important because *VPC peering is region-dependent* and once deployed, you cannot change the region.

- ##### Instance Size
Fourth option **Instance Size**, this is important too. If you are trying out Atlas for the first time or want to test a development version choose **M0**, which is a free cluster offered by the mongoDB. But for production workloads, Atlas recommends **M30** & higher versions. When you select it, you can also see the cost associated with it. The **Cost** is shown for a *single cluster(three replicas)*. Once you select the version say **M30**, you will have to select the **RAM**, **storage capacity**, **storage speed** and **Encryption** option.

- ##### RAM
MongoDB makes **extensive** use of RAM to speed up database operations. In MongoDB, all data is read and manipulated through in-memory representations of the data. Reading *data from memory* is measured in
`nanoseconds` and *reading data from disk* is measured in `milliseconds`, thus reading from memory is orders of magnitude faster than reading from disk.

*The set of data and indexes that are accessed during normal operations is called the working set*. It is best practice that the working set fits in RAM. It may be the case the working set represents a fraction of the entire database, such as in applications where data related to recent events or popular products is accessed most commonly.

*When MongoDB attempts to access data that has not been loaded in RAM, it must be read from disk. If there is free memory then the operating system can locate the data on disk and load it into memory directly.* However, if there is no free memory, MongoDB must write some other data from memory to disk, and then read the requested data into memory. This process can be time consuming and significantly slower than accessing data that is already resident in memory.

Some operations may inadvertently purge a large percentage of the working set from memory, which adversely affects performance. For example, a query that scans all documents in the database, where the database is larger than available RAM on the server, will cause documents to be read into memory and may lead to portions of the working set being written out to disk. Other examples include various maintenance operations such as compacting or repairing a database and rebuilding indexes.

If your *database working set size exceeds the available RAM of your system, consider provisioning an instance with larger RAM capacity (scaling up) or sharding the database across additional instances (scaling out)*. Scaling is an automated, on-line operation which is launched by selecting the new configuration after clicking the CONFIGURE button in MongoDB Atlas. It is easier to implement sharding before the system’s resources are consumed, so capacity planning is an important element in successful project delivery.

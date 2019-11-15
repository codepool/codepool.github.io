---
layout: post
title:  "Building a Real World app with Neo4j"
date:   2018-10-02 17:57:22 +0530
categories: jekyll update
comments: true
description: "Learn how to use Neo4j graph database to build a content platform like Medium"
---
Most of the software applications today are powered by relational databases as they provide an easy way for storing structured data that fit well into tables. But as the data beings to grow, a relational databases fail to provide scale due to their rigidity as they store data in a very organized manner with predefined columns. Joins on the tables result in slow responses as table sizes become large. NoSQL databases were created to store unstructured data and query them efficiently. 
<br/>
Neo4j is a graph NoSQL database which is best suited for highly interconnected data. Since a graph database works by storing the relationships along with the data, it is useful for building applications like fraud detection or recommendation engines which require not just knowing about one entity but also how the entity is connected to other entities.

### Building a graph for Medium using Neo4j

Let's understand the power of a graph database by building a real world application like the popular content platform - Medium. Although Medium supports many features, we'll try to build only the core functionalities just to understand the use of Neo4j. A Medium user can perform the following actions :

* User can follow a topic
* User can follow another user
* User can publish a story
* Story can be tagged with one or more tags
* User can clap and/or respond on a story

### Creating Nodes and Relationships

In graph world we represent the nouns like User, Story, Topic, Tag as **nodes** while the verbs like PUBLISHED, FOLLOWED, RESPONDED, CLAPPED, TAGGED_WITH etc represent **relationship** between the nodes for eg User **PUBLISHED** Story shows that a relationship PUBLISHED exists between nodes User and Story. Both nodes and relationships can store properties of their own.
Let's create User, Topic and Tag nodes for our application.

{% highlight javascript linenos %}
const neo4j = require('neo4j-driver').v1
const driver = neo4j.driver("bolt://localhost", neo4j.auth.basic("neo4j", "password"));
var session = driver.session();

var cypher = 'create(u:User{name:{name}, username: {username}, email:{email}})
create (:Topic{name: {topic}})
create (:Tag{name: {tag}})';
Promise.all([
	session.run(cypher, {name: 'Alan', username: 'alan', email: 'alan@gmail.com', topic: 'Programming', tag: 'Neo4j'}),
	session.run(cypher, {name: 'Dave', username: 'dave', email: 'dave@gmail.com', topic: 'Javascript', tag: 'Scalability'})
])
.then((results) => session.close())
.catch((error) => session.close());

{% endhighlight %}

So we have created two users with properties - name, email, username. **:User**, **:Topic**, **:Tag** are actually labels given to the 3 nodes. Note that we have only created the nodes and not the relationships. Let's create relationships bewteen these nodes

{% highlight javascript linenos %}

var cypher = `match(u1:User{name:'Alan'), (u2:User{name:'Dave'), 
(t1:Topic{name:'Programming'}), (t2:Topic{name:'Javascript'}), 
g1:Tag{name:'Scalability'}), (g2:Tag{name:'Neo4j'})
create (u1)-[:FOLLOWS]->(u2)
create (u1)-[:FOLLOWS]->(t1)
create (u2)-[:FOLLOWS]->(t2)
create (u1)-[:PUBLISHED]->(s1:Story{title:'Scaling web apps'})-[:TAGGED_WITH]->(g1)
create (u1)-[:PUBLISHED]->(s2:Story{title:'Neo4j is awesome'})-[:TAGGED_WITH]->(g2)
create (u2)-[:PUBLISHED]->(s3:Story{title:'Neo4j is powerful'})-[:TAGGED_WITH]->(g2)
create (u1)-[:CLAPPED]->(s3)
create (u2)-[:RESPONDED]->(s1)
`
session.run(cypher)
.then((results) => session.close())
.catch((error) => session.close());

{% endhighlight %}

We have created relationships and few more nodes. A node is written inside parentheses () and relationships in square brackets []. In Neo4j we can add nodes and relationships dynamically to the existing graph. We can even modify or add properties of nodes and relationships. Our graph will now look like the following which is very easy to visualize how the entire data is connected. 


![image](/assets/img/neo4j.png)

Circles represent the nodes and arrows represent the relationships. Once you have built up the graph, navigating through the graph and retrieving nodes becomes very easy.

### Querying the graph with Cypher

Now that we have our graph ready, querying it is fairly easy using Cypher - Neo4j's query language. When you visit a user's Medium profile, it shows you the stories on which the user clapped or responded. Let's query this data for our user **Dave**

{% highlight javascript linenos %}

var cypher = `match (dave:User{name:"Dave"})-[:CLAPPED|RESPONDED]->(story:Story)
return story`
session.run(cypher)
.then((results) => session.close())
.catch((error) => session.close());

{% endhighlight %}

Now, let's get all the stories published by Dave along with the number of claps and responses for each story.

{% highlight javascript linenos %}

var cypher = `match (:User{name:"Dave"})-[:PUBLISHED]->(story:Story)<-[r:RESPONDED|CLAPPED]-(:User)
return story, TYPE(r), count(r)`
session.run(cypher)
.then((results) => session.close())
.catch((error) => session.close());

{% endhighlight %}

Piece of cake, isn't it? No complex joins or confusing GROUP BY clauses. Let's look at a more complex query.

### Recommending stories to users

One of the most important features of content platforms like Medium is **content discovery** - suggesting new content that the user may be interested in which otherwise may left undiscovered. Medium shows relevant stories to a user out of the thousands stories that are published each day based on some ranking algorithm. For eg it can give a higher score to a story if it's clapped or responded by your follower/followee, hence it may appear higher on your homepage.

Let's implement a hypothetical complex scoring criteria which might help in getting the most relevant stories.
Let's say Medium wants to show our user **Dave**, stories published by other users whom Dave does not follow but had responded or clapped on their stories in the past and the stories carry tags which Dave's one or more stories also carry.

Let's break this criteria into pieces :

* Stories are published by users whom Dave does not follow - *implying the users are currently undiscovered by Dave*
* Dave has responded or clapped on one or more stories by these users in the past - *implying Dave has shown some interest in the articles these users write*
* Stories contain tags that Dave's stories also contain - *implying stories share same interest as Dave*

Implementing such a solution using relational database will require joins between following tables - users, stories, tags, claps, responses, followers which will be quite slow if the user base is huge. The query itself will be quite complex to write. 

But using Neo4j implemeting this is fairly easy using **Cypher**.

{% highlight javascript linenos %}

var cypher = `match (dave:User{name:"Dave"})-[:PUBLISHED]->(:Story)-[:TAGGED_WITH]->(dave_tag:Tag), 
(user:User)-[:PUBLISHED]->(story:Story)-[:TAGGED_WITH]->(tag:Tag) 
where user.name <> "Dave"
and not (dave)-[:FOLLOWS]->(user)
and not (dave)--(story)
and (dave)-[:CLAPPED|:RESPONDED]->(:Story)<-[:PUBLISHED]-(user)
and dave_tag.name=tag.name
return story`
session.run(cypher)
.then((results) => session.close())
.catch((error) => session.close());

{% endhighlight %}

So whenever you feel the need to store and inspect your data which is heavy on it's relationship with other data, try using a graph database like Neo4j. Not only will you be able to query the data faster but also visualize the data in more intuitive way.


<br><br>
{% include disqus.html %}
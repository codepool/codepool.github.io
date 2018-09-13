---
layout: post
title:  "Rate Limiting using Redis Lists and Sorted Sets"
date:   2018-06-27 17:57:22 +0530
categories: jekyll update
---
Rate limit is the process wherein you restrict (to some reasonable value) your users to access an API or service to prevent malicious requests bring down your servers. While the basic rate limiting can be easily configured on load balancers like Nginx, you might want to consider implementing your custom solution for a more specific business use case for eg. restricting a user to do only 10 transactions using their credit card in a day. Following are some of the use cases that need rate limiting :

1. Free SMS service might restrict a user to send only 10 SMS per day.
1. Many services such as Google APIs put rate limit on their API usage per day.
1. A banking app might allow a user to transfer a total sum of Rs 50,000 per day especially if the payee account number has been recently added. Typically this is to prevent fraud.


Today we'll learn how to implement these features using Redis. While Redis is popularly known for appplication caching, it has powerful built-in data structures that can do wonders.

### Allow only 'N' API requests per minute on a running window

```javascript
var redis = require("redis"),
client = redis.createClient();
```


```java
var redis = require("redis"),
String s;
```
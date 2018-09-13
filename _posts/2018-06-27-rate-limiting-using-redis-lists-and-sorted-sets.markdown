---
layout: post
title:  "Rate Limiting using Redis Lists and Sorted Sets"
date:   2018-06-27 17:57:22 +0530
categories: jekyll update
comments: true
---
Rate limiting is the process where you restrict your users to access an API or service to prevent too many malicious requests bring down your servers. While the basic rate limiting can be easily configured on load balancers like Nginx, you might want to consider implementing your custom solution for a more specific business use case. Following are some of the use cases that need rate limiting :

1. Free SMS service might restrict a user to send only 10 SMS per day.
1. Many services such as Google APIs put rate limit on their API usage per day based on API key.
1. A banking app might allow a new user to transfer a total sum of  $2,000 per day.

Today we’ll learn how to implement these features using Redis. While Redis is popularly known for application caching, it has powerful built-in data structures that can be leveraged for many other use cases.

### Allow only ‘N’ API requests per minute
Let’s assume we have an api /products to get the list of products on your website. If you do not want any DDoS attacks upon this API you need to put some kind of rate limiting to prevent unlimited access. Let’s say we want to serve only 5 requests to this API call in a minute. Below is the **NodeJS** code to implement this.

{% highlight javascript linenos %}
const redis = require("redis");
const redisClient = redis.createClient();

const lua = `if redis.call('EXISTS', KEYS[1]) == 0 
	then 
		redis.call('SETEX', KEYS[1], 60, 0) 
	end
	redis.call('INCR', KEYS[1])
	if tonumber(redis.call('GET', KEYS[1])) <= 5
	then return 'pass'
	else return 'exceeded limit'
	end`;

redisClient.eval(lua, 1, 'products', function(err, replies) {
	// replies will hold the data returned by Lua.
});

{% endhighlight %}

We have used Lua script to execute redis commands with the help of **redis.call()** function. **eval()** is a function provided by Redis to execute the complete Lua script. 2nd and 3rd arguments of eval function are the number of redis keys and the actual key names. We are storing the count of api hits in **products** key. The idea is to create the key with auto-expity of 60 seconds and increment the counter every time the API is accessed. Also note that this code is thread safe and atomic in nature. Redis won’t execute this script from another parallel client until it finishes execution from the current client.

But do you see a drawback in this approach? This code works only on a calendar minute and not on a rolling window. Which means if the server gets 5 hits at 10:59 an attacker can do 5 more hits at 11:01 which means 10 hits in just 2 seconds. Clearly we need a mechanism to put rate limits on a rolling 1 minute window. We will use Redis List data structure to implement this feature.

### Allow only 'N' API requests per minute on a running window

{% highlight javascript linenos %}
const redis = require("redis");
const redisClient = redis.createClient();

const lua = `local count = tonumber(redis.call('LLEN', KEYS[1]))
	if count < 5 
	then	
		redis.call('LPUSH', KEYS[1], ARGV[1]) 
		return 'pass'
	else
		local time = tonumber(redis.call('LINDEX', KEYS[1], -1))
		if ARGV[1] - time <= 60 * 1000 
		then 
			return 'exceeded limit'
		else 
			redis.call('LPUSH', KEYS[1], ARGV[1])
			redis.call('RPOP', KEYS[1])
			return 'pass'
		end	
	end`;

redisClient.eval(lua, 1, 'products', Date.now(), function(err, replies) {
	// replies will hold the data returned by Lua.
});
{% endhighlight %}

We have stored current time in milliseconds in a Redis list in a sorted manner with the newest items pushed to the front of the list. As soon as the list crosses count 5, we analyze the list taking the last (oldest) element from the list and comparing it with current time. If 1 minute has crossed since we inserted the element, we allow access to the user and remove the element.


### Rate Limiting using Sorted Sets
Let’s achieve the rolling window rate limiting using another data structure - Sorted Set. Redis sorted set is a collection of non repeating elements where each element can be assigned a score which is any number. Then you can perform operations like getting all elements between two scores or deleting elements between two scores in O(log N). A classic application of sorted set is to assign current timestamp to each element as it’s score and then perform range queries between two timestamps. Let’s solve our rate limiting problem using this approach.

{% highlight javascript linenos %}
const redis = require("redis");
const redisClient = redis.createClient();

const lua = `redis.call('ZREMRANGEBYSCORE', KEYS[1], 0, ARGV[1] - 60 * 1000) 
	if tonumber(redis.call('ZCARD', KEYS[1])) < 5
	then	
		redis.call('ZADD', KEYS[1], ARGV[1], ARGV[1])
		return 'pass'
	else
		return 'exceeded limit'
	end`;

redisClient.eval(lua, 1, 'products', Date.now(), function(err, replies) {
	// replies will hold the data returned by Lua.
});
{% endhighlight %}

The idea is to remove all elements that have exceeded the 1 minute window period and then take the count of rest of the elements.

### Why did we use Lua instead of MULTI/EXEC?
Well, Redis is a basic data structure store and does not support conditional statements like if…else. Also we needed to make sure that our code runs properly in a multi-threaded environment. Using Lua makes sure that no other redis client can execute the script parallelly. We could have used **MULTI/EXEC** commands in Redis but the conditional statements will still need to be outside of redis environment which may not allow our program to give proper results in a multi threaded environment.   

<br><br>
{% include disqus.html %}
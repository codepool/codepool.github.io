---
layout: post
title:  "From JS Callbacks to Promises to Async/Await"
date:   2019-02-15 18:31:12 +0530
categories: jekyll update
comments: true
---

In the recent years, Javascript as a language has evolved a lot making the development much easier than before. New versions of EcmaScript (ES) are being released almost every year bringing in more and more improvements to the language.

Let’s understand by examples how asynchronous programming in Javascript has evolved in the past years. In the below examples, I’ll use **setTimeout()** function to simulate long running asynchronous tasks.

### Using Callbacks
Javascript developers coding before the ES6 release would understand the pain of writing asynchronous Javascript code using callbacks.
Here is a simple example demonstrating the use of callback functions :

{% highlight javascript linenos %}
const first = (value, callback) => {
    setTimeout(() => {
        callback(value + 2)
    }, 1000)
}

const second = (value, callback) => {
    setTimeout(() => {
        callback(value + 2)
    }, 1000)
}

const third = (value, callback) => {
    setTimeout(() => {
        callback(value + 2)
    }, 1000)
}

const main = () => {
    first(5, (firstResult)=> {
        second(firstResult, (secondResult) => {
            third(secondResult, (thirdResult) => {
                console.log(thirdResult);
            })
        })
    });       
}
main();
console.log("end program");

{% endhighlight %}

The above code will output “end program” first and then “16” after few seconds. This is because setTimeout() does not block the code and all the functions return immediately resulting in last line of the code to be printed first. Since the third() function is dependent on second() function result hence we are bound to call it inside the body of third() function.

Notice that due to so many callbacks, the code has become ugly and less readable. This is what we refer to as **Callback Hell**.

### Using Promise
ES6 introduced Promise in 2015. Promise is nothing but an object that may output a value in future. We can rewrite above code eliminating any callbacks like below:

{% highlight javascript linenos %}

const first = (value) => {
    return new Promise(resolve => setTimeout(() => resolve(value + 2), 1000));
}

const second = (value) => {
    return new Promise(resolve => setTimeout(() => resolve(value + 2), 1000));
}

const third = (value) => {
    return new Promise(resolve => setTimeout(() => resolve(value + 2), 1000));
}

const main = () => {

    const initialValue = 10;
    first(initialValue)
    .then(firstResult => {
        return second(firstResult)
    })
    .then(secondResult => {
        return third(secondResult);
    })
    .then(thirdResult => {
        console.log(thirdResult);
    }) 
}
main();
console.log("end program");

{% endhighlight %}

Much cleaner! Instead of writing function inside function, we are chaining the function calls using promises then(). We can convert any function taking a callback as argument into returning a Promise. Instead of calling the callback function, the function can call resolve() method to return the value in future.

With Promises, although the code becomes more readable, still it feels as if we are developing and reading our code in an asynchronous manner.

### Using async/await

ES8 introduced async/await keywords that now let developers write code in a more cleaner sequential manner. Let’s convert our above code into async/await.

{% highlight javascript linenos %}
const first =  (value) => {
    return new Promise(resolve => setTimeout(() => resolve(value + 2), 1000));
}

const second = (value) => {
    return new Promise(resolve => setTimeout(() => resolve(value + 2), 1000));
}

const third =  (value) => {
    return new Promise(resolve => setTimeout(() => resolve(value + 2), 1000));
}

const main = async () => {
    const initialValue = 10;
    const firstResult = await first(initialValue);
    const secondResult = await first(firstResult);
    const thirdResult = await first(secondResult);
    console.log(thirdResult);
}
main();
console.log("end program");

{% endhighlight %}


The output and execution order of the code remains the same i.e “end program” gets printed first and then “16” after few seconds. But notice our code looks very clean and it feels as if we are writing code in synchronous way.
Note that, async is not a replacement of promises. In fact, async functions automatically return promise even though you don’t specify it explicitly.
await blocks the code execution within the async function in which it is used. Since it is mandatory to declare such function as async, the entire function itself executes asynchronously, that’s why our main() function returns immediately without blocking the code.


### Node util.promisify
Many of the exiting JS library functions/modules still use the old good callbacks and do not return promises. If you are using such functions into your code how do you use Promise then() or await with such functions?
Fortunately, Node v8 introduced **promisify()** method in the in-built **util** library which takes any function as its argument and returns a Promise. Let’s take a look at below example:

{% highlight javascript linenos %}
const myFunction = (value, callback) => {
    callback(value);
}

//above function can be converted to a promise returning function like below

const util = require('util');
const myFunctionAsync = util.promisify(myFunction); //myFunctionAsync now returns a Promise

//now you can use this function like :

myFunctionAsync
.then(result => {

})

//or using ES8 await like 

const result = await myFunctionAsync();

{% endhighlight %}


Some of the functions in node **fs** module like **readFile()** accept callback as argument and can be converted to **Promise** using above technique.
Async/await let developers write code in a synchronous manner, while keeping the actual execution still asynchronous. It allows debugging the code and include exception handling more easily.

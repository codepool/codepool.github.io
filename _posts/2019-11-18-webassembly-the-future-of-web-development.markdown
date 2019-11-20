---
layout: post
title:  "WebAssembly - The future of Web development"
date:   2019-11-18 11:25:00 +0530
categories: jekyll update
comments: true
description: "Learn how WebAssembly is going to shape the future of Web development"
---

## What is WebAssembly?

WebAssembly (wasm) is a new binary instruction format that promises near-native performance for web applications. Simply put, one can develop code in high level languages which can then be compiled to WebAssembly module (wasm) and can be directly executed by modern browsers. I like a bytecode that can run on web browsers.

For many years, the only language available in the market for programming web applications was JavaScript. No other language could run inside the browser. But Javascript has many drawbacks for eg it’s slow and has type safety issues. Javascript was infact never designed to be used for heavy computation applications. To overcome some of the JS drawbacks, many developers started building transpilers to convert other languages (like CoffeeScript, TypeScript etc) to JavaScript. Still the final code running was still JavaScript. WebAssembly was created to sort out JS limitations and it is proving to be a revolutionary step towards the future of web development.

## What WebAssembly is Not?

WebAssembly, on its own, isn’t a programming language. Instead developers write program using a high level language like Rust or C++ and compile it into binary .wasm file

WebAssembly is not a replacement for Javascript. In fact, it is supposed to be run alongside javascript. For example, you may choose to build only high computation code in WebAssembly and use it alongside with other light weight JavaScript code. You also need Javascript to load wasm modules in browser.

## WebAssembly Architecture

WebAssembly code is designed to run on stack-based virtual machine. Unlike a register machine where the operands lie on CPU registers and computations happen there itself, in a stack machine, most of the instructions assume that the operands are sitting on the stack, rather than stored in specified registers. Let’s take this instruction for example:

**push A**<br>
**push B**<br>
**add**

To add two numbers in a stack machine, you push those numbers onto the top of the stack. Then you push the ADD instruction onto the stack. The two operands and the instruction are then popped off the top and the result of the addition is pushed on in their place. There are some well-known stack machines like JVM, .NET Runtime etc. 

WebAssembly does not directly interact with the OS. In fact, it's sandboxed and executed inside the same JS engine. WebAssembly doesn’t have a heap in the traditional sense. There’s no concept of a new operator. There’s also no garbage collection. Instead, WebAssembly has linear memory i.e memory is represented as a contiguous range of untyped bytes. Your WebAssembly module can grow the linear memory block in increments called pages of 64KB if it needs more space.
WebAssembly has only four data types: **i32, i64, f32, f64** for 32 & 64-bit integers & floating point numbers

## Building a WebAssembly application

WebAssembly has both a text format (**.wat**) and binary format (**.wasm**). Binary format is the one that runs on browser. Although we can code .wat and compile it to .wasm, it’s not a desirable approach as it's a pretty low-level language. We would normally compile a high level language to .wasm. But, just to understand the internals, let’s start by writing some code in wat.

### Using .wat

{% highlight webassembly linenos %}
(module
  (func $add (param $lhs i32) (param $rhs i32) (result i32)
    local.get $lhs
    local.get $rhs
    i32.add)
  (export "add" (func $add))
)

{% endhighlight %}

The fundamental unit of code in WebAssembly is a module. Inside the module, we declared one function add() which takes two input parameters of data type i32. At the end we export this method so that it can be consumed by external Javascript code
We can compile the above .wat file into .wasm file using WebAssembly toolkit (**wat2wasm**). We can then load the wasm file using Javascript like below:

{% highlight html linenos %}

<html>
    <script>
        WebAssembly.instantiateStreaming(fetch('demo.wasm'))
        .then(obj => {
            const result = obj.instance.exports.add(3,10);
            alert(result)
        });
    </script>
</html>

{% endhighlight %}


### Using Rust

Let’s create a simple WebAssembly module to compute factorial using Rust language. But first, we need to install rustup from <https://rustup.rs>. This will install rustup, cargo & rustrc. Change the default toolchain to nightly <span style="background:#eef; padding:1px 5px; border:1px solid #eee">rustup default nightly</span>. Next, we will add the WebAssembly target using command <span style="background:#eef; padding:1px 5px; border:1px solid #eee"> rustup target add wasm32-unknown-unknown</span>. We can verify the installation using command <span style="background:#eef; padding:1px 5px; border:1px solid #eee"> rustup target list</span>. Next create a Rust project using command <span style="background:#eef; padding:1px 5px; border:1px solid #eee"> cargo new --lib rustdemo</span>. This will create Cargo.toml & src/lib.js inside rustdemo folder. Edit Cargo.toml and add following line :
​
{% highlight html%}
[lib]
​crate-type = ​["cdylib"]
...
[dependencies]
{% endhighlight %}

The above lines mean you are telling the compiler to create a dynamic system library depending upon the target. For WebAssembly target it will just create a *.wasm file. For other targets it will create *.so file on Linux, *.dll Windows etc
Next we will write code to compute factorial inside src/factorial.rs


{% highlight rust linenos %}

pub fn factorial(n: i32) -> i32 {
  if n == 0 {
      return 1;
  }
  n * factorial(n - 1)
}

{% endhighlight %}

Now let’s create the WebAssembly interface in src/lib.js and expose the method to be consumed by external JS code

{% highlight rust linenos %}

mod factorial; //import factorial module
#[no_mangle]
pub extern "C" fn compute_factorial(num: i32) -> i32 {
        factorial::factorial(num)
}

{% endhighlight %}


Now let’s build wasm file using command <span style="background:#eef; padding:1px 5px; border:1px solid #eee"> cargo build --target wasm32-unknown-unknown </span> from inside the rustdemo folder. This will create wasm32-unknown-unknown/debug/rustdemo.wasm inside the target folder.

Finally, let’s write our front-end html code to loan our wasm module

{% highlight html linenos %}
<html>
    <script>
        WebAssembly.instantiateStreaming(fetch('rustdemo.wasm'))
        .then(obj => {
            const result = obj.instance.exports.compute_factorial(10);
            console.log(result)
        });
    </script>
</html>

{% endhighlight %}

Bingo! We have written our first WebAssembly module using Rust programming language. You can check the performance of the above code compared to a raw JS code for factorial. You will see for yourself how fast WebAssembly is!

## Running WebAssembly outside the browser

While WebAssembly was primarily designed to run on browsers, however it can run outside the browsers too. For eg. you can execute the above wasm file using Node JS like

{% highlight javascript linenos %}
const fs = require('fs');

const demo = async () => {
  const buffer = fs.readFileSync("./rustdemo.wasm");
  const result = await WebAssembly.instantiate(buffer);
  console.log(result.instance.exports.compute_factorial(10));
};

demo();

{% endhighlight %}

You can also run wasm module without browser using **WebAssembly System Interface** (wasi) complaint runtime like **wastime**. wastime is available both as a command line tool and library which can allow loading of wasm files inside python, nodejs etc. To use wasi, the wasm file should be created using a different build target eg <span style="background:#eef; padding:1px 5px; border:1px solid #eee">rustup target add wasm32-wasi</span> and <span style="background:#eef; padding:1px 5px; border:1px solid #eee">cargo build --target wasm32-wasi</span>


## Advantages of WebAssembly

* WebAssembly module is much faster than the corresponding Javascript code . Since wasm binary files are much smaller in size compared to textual JS files they can be downloaded over the internet faster by the browser. Also since they are compiled, the performance is much better than the interpretted JS code. In fact WebAssembly is believed to be only around 10% slower than native machine code which is astonishing.

* It offers a good compile target for the web, so that people can choose whatever language they want to code their websites into.

## How WebAssembly will shape the future of the Web?

Right now you might be thinking why would someone want to ditch a language as simple as Javascript and start coding in harder to understand languages like C/C++/Rust. Well, WebAssembly is quite new right now with not enough community around it. But think of all the high-performance video and photo transcoding libraries, any graphics or physics engine that uses OpenGL, application frameworks like Qt etc. These will likely be made available via WebAssembly modules once the community around it starts growing. WebAssembly can be revolutionary for the game development companies!

The heavy performance gains WebAssembly provides in the code execution could one day see the heaviest of desktop software running inside the web browser. WebAssembly might literally make the native desktop softwares dead in coming years.

<br><br>
{% include disqus.html %}





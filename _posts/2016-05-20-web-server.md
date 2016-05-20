---
layout: post
title: Let's Build a Web Server in Rust
tag: rust-server
---

#### Where do we begin?

Well I know that a web server basically sets up a connection to a network port and then recieves requests via text and then responds via text using the HTTP protocol. Lets define what a basic request and response looks like so that we can shoot for that when building our server. Here is a basic HTTP request and response.

Request:

```
GET / HTTP/1.1
Host: www.example.com
```

Response:

```
HTTP/1.1 200 OK

Hello, World!
```

So what will we need to start? Some way to connect to a port, a way to listen for incoming requests on that port, and then a way send something back. 

#### Listen and Learn

So let's work on setting up the port first. In Rust most of the code to setup a network connection lives in the [`std::net`](https://doc.rust-lang.org/std/net/index.html). We'll be using `TcpListener` and `TcpStream` from this module. Thankfully the Rust docs for the `TcpListener` give us a helping hand with how to use it. You might notice the `Tcp` on the module names, this is type of protocol we are using to communicate through the ports, in fact most of communication on the web uses either [`TCP` or `UDP`](https://en.wikibooks.org/wiki/Communication_Networks/TCP_and_UDP_Protocols). I've rewritten the code to make it a bit more simple but I'll annotate as I go so we can see what everything does.

```rust
// In Rust we need to tell it where things are from, 
// in this case we are using the read_to_string method
// so we need to bring in the std::io::Read
// module to the party. We also need TcpListener and
// TcpStream
use std::io::Read;
use std::net::{TcpListener, TcpStream};

fn main() {
    // bind allows us to create a connection on the port
    // and gets it ready to accept connections.
    let listener = TcpListener::bind("127.0.0.1:5432").unwrap();
    
    // The listener's accept method waits or 'blocks' until
    // we have a connection and then returns a new TcpStream
    // that we can read and write data to.
    let stream = listener.accept().unwrap().0;
    read_request(stream);

    // This will close the listener and exit the program
    drop(listener);
}

// This function takes the stream we just got from the
// listener and then reads some data from it.
fn read_request(mut stream: TcpStream) {
    let mut request_data = String::new();

    // The read_to_string method uses the string we pass
    // in and fills it up with the data from the stream.
    stream.read_to_string(&mut request_data);

    // Finally we print the data
    println!("{}", request_data);
}
```

If you copy this code into a new Rust project and run it you can start the server and hit [localhost:5432](http://localhost:5432) in your browser. Unfortunately for us our program doesn't know when to stop reading from the `TcpStream` we created and our connection with the browser will run until it times out. Once we stop the request on the browser the output from our program will display. This is pretty inconvienient. Let's try to fix this!

#### Let's Write - A Correspondence

In HTTP/1.1 the browser doesn't just close the connection to the server when it's done writing, it's expecting us to know when it's done talking and give it a response on the same connection. A few things have to change to let this happen. First we can't wait until the browser times out to read the output of the `TcpStream`. We'd also like to be able to give the browser a response back. In order to do that we need to write on the stream. Let's see what's changed. 

```rust
use std::io::{Read, Write, BufReader, BufRead};
use std::net::{TcpListener, TcpStream};
```
We added a few more modules up here. `Write`, `BufReader`, and `BufRead`. `Write`, like the name suggests allows us to write to our stream. `BufReader` and `BufRead` let us use our stream more effectively.
<br>
<br>

```rust
// Everything is the same in here
fn main() {
    let listener = TcpListener::bind("127.0.0.1:5432").unwrap();
    
    // The .0 at the end is indexing a tuple, FYI
    let stream = listener.accept().unwrap().0;
    handle_request(stream);

    drop(listener);
}

// Things change a bit in here
fn handle_request(stream: TcpStream) {

    let mut reader = BufReader::new(stream);

    for line in reader.by_ref().lines() {
        if line.unwrap() == "" {
            break;
        }
    }

    send_response(reader.into_inner());
}
```

Instead of just dumping everything into a string we want to be able to read and process the data being written into the stream by the browser. To do this we're using the `BufReader`, so buff! Really it has an internal byte buffer so we can read from the buffer instead of the stream which makes it easier to work with. Because of `BufRead` at the top we can use the `lines()` method to pluck off lines from the browser request. We need `by_ref()` here otherwise Rust will take the reader, and not let us use it later on. Basically the `lines()` method is borrowing the `BufReader` instead of stealing it away forever. In 'real-life' we would actually do something with the contents of the headers but for now we are just waiting until we see a blank line to stop reading from the stream.
<br>
<br>

```rust
// New function to write back with!
fn send_response(mut stream: TcpStream) {
    // Write the header and the html body
    let response = "HTTP/1.1 200 OK\n\n<html><body>Hello, World!</body></html>";
    stream.write_all(response.as_bytes()).unwrap();
}
```

This one is pretty simple. We write the data we want that contains the HTTP header and the response body, in this case static HTML. HTTP needs those `\n\n` in the middle to know when the headers are done and when to interpret the response body. Finally we write the response to the stream. You may have noticed the `unwrap()`'s. Rust returns a [`Result`](https://doc.rust-lang.org/book/error-handling.html#the-result-type), which in production Rust code we would catch and deal with, but for now we don't need to worry about it, because we are learning and we're not in production. Finally we can run our updated code, hit the browser and see our response show up in our browser window! We did it! 😃

#### What's Next (Troubles, Additions)

I mostly wrote this to document what it was like to build a simple web server and at least get acquainted with Rust's networking API. A lot of this code is not what you want to write if your life depended on it. More just for fun. A few of the things that were hard for me, was trying to use the iterator over the `BufReader` in the `for line in reader.by_ref().lines()` part. Without the `by_ref()` Rust thinks we are done with the `BufReader` and wouldn't let us use it anymore. Another thing I had trouble with was that the server would sit waiting for more data from the connection with the browser. In the past, I think in `HTTP/1.0`, the browser would automatically close the connection after sending the response, so you don't have to check for the empty string. So I had to think of a way that would work. I still don't know exactly what the 'right' way to do it is. It's actually [pretty complicated](https://tools.ietf.org/html/rfc2616#section-19.3). 

Some more stuff you might want to add to make the server better is to actually parse the headers and get the actual resource requested and return it based on the request path, another thing that we could do is add multithreading and start a new thread for each server connection and actually take more than one request without restarting the server 😜. Adding [`HTTP/2`](http://httpwg.org/specs/rfc7540.html) support would also be an interesting idea. Although not very fancy or robust this is a basic server that you can play around with in the browser and maybe learn a little bit about HTTP and Rust's net code in the process. Thanks for reading!
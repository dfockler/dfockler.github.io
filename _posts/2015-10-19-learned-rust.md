---
layout: post
title: Things I learned from Rust
tag: learned-rust
---
When you learn a new programming language, a lot of times you will come across concepts or ideas that make you realize those ideas exist in other languages. In Rust there are a lot of these ideas, and while learning Rust I've come across quite a few of these. Some other readers may have realized these right away or may have already known about them from other languages. I think it's really awesome when I find some cool idea and then find it pop up in another place. Here are some of the ones I've found in Rust.



####Handling Uncertainty with Options

One of the cool features of Rust is the concept of handling code whereby you are guaranteed to get a type back that you can deal with. `Option` is an enum with two possible values, `Some` or `None`. Meaning if your function takes an `Option` you will either get back a `Some` with a value or `None`.

{% highlight rust %}
fn cookie_jar(value: i32) -> Option<i32> {
    let cookie_count = 13;
    
    if value < cookie_count {
        Some(cookie_count - value)
    } else {
        None
    }
}

fn main() {
    match cookie_jar(14) {
        Some(t) => println!("{} cookie(s) left", t),
        None => println!("No more cookies")
    }
}
{% endhighlight %}

Because our `cookie_jar` function returns an `Option`, we know we will either get back a `Some` or a `None` and we can use a `match` to run code depending on which situation we get. In fact Rust won't let us not account for the `None` situation. If our `match` instead looked like this

{% highlight rust %}
match cookie_jar(14) {
    Some(t) => println!("{} cookie(s) left", t)
}
{% endhighlight %}

Rust would return an error on compilation. 

```
error: non-exhaustive patterns: `None` not covered
```

Rust makes sure we've accounted for all the situations that could happen given the `Option` type that was returned. The underlying way this happens is through enums, which specify variants that the `match` must handle. There are many more examples of types like this in Rust. If you want to know more check out the [Error Handling Section](https://doc.rust-lang.org/stable/book/error-handling.html#handling-errors-with-option-and-result)  of the Rust Book. This concept is also built into the Haskell language, where it is known as the `Maybe` type.

#### Pattern Matching

Another concept that relates to `match` is how the `match` is able to actually match against anything. Rust uses pattern <i>match</i>ing to match the variable to the correct branch, hence the name `match`. Pattern matching is a really important idea in other languages. In Elixir and Erlang pattern matching even goes so far as to be used to match which function to call. Rust can match in a number of different ways.

{% highlight rust %}
match x {
    1 => (),            //numeric values
    1 ... 20 => (),     //numeric ranges
    'a' => (),          //char values
    'a' ... 'f' => (),  //char ranges
    "bob" => (),        //strings
    expr | expr => (),  //something or something else
    Some(value) => (),  //types
    _ => (),            //Anything else
}
{% endhighlight %}

Those are just a few of the nice patterns that you can use to match with in Rust. To see more ways to match with examples check out the Rust book section on [Patterns](https://doc.rust-lang.org/stable/book/patterns.html).

#### Enums

You may have already heard of enums if you are used to using a SQL database or another statically typed language. For people like me who haven't had much experience with them, they are a useful tool to organize your code and represent the state of various parts of your code. Enums are a definition of the variants of a type. That sentence doesn't really do enums justice. Really they basically limit what versions a type can be, and what data that version can contain, while still being a type. That still doesn't really make sense, but lets show an example.

{% highlight rust %}
enum Bicycle {
    Fixie,
    Road(i32),
    Recumbent { flag_color: Color, recline_angle: i32 },
}

fn speed(bike: Bicycle) {
    match bike { //Throw in some previous concepts
        Bicycle::Fixie => println!("{} mph", 30),
        Bicycle::Road => println!("{} mph", 25),
        Bicycle::Recumbent => println!("{} mph", 40), 
    }
}
{% endhighlight %}

Here we have the Bicycle enum. Thanks [Sandi Metz](http://www.poodr.com/) :P The `Bicycle` type can be used anywhere a normal type would be used, but hidden inside of it, is another type, the `Bicycle`'s variant. In this case the Bicycle could be a Fixie, a Road Bike, or a Recumbent Bike. In Rust your enums can also store data for instance the Road Bike would store the data for it's gear ratio. You can also store named data. In the Recumbent Bike we have data for the color of the flag on the back of the bike, as well as the reclining angle of the seat. Enums in Rust are a nice way to carry around extra information for types that are mostly similar but could be slight <i>variants</i> of each other.

#### Dangling Commas

Just a quick FYI, Rust allows for lists to have dangling commas, which are a sublime syntactic quality. :D
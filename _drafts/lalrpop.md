---
layout: post
title: Let's Build a REPL/Parser with Rust & LALRPOP
tag: lalrpop
---

A lot of this post is kind of a pure ripoff of the [LALRPOP Tutorial](https://github.com/nikomatsakis/lalrpop/blob/master/doc/tutorial.md) which was written by an actual professional in language parsing. This is sort of my layman's interpretation of some of the concepts and ideas, so I can learn and share hopefully correctish information and maybe get some other people to try stuff out that might seem scary at first. My intention is that you can get a base of knowledge and code to help you expand the ideas further and build your own cool stuff.

#### A Highspeed Flyby of Programming Language Parsing

So first things first, we have to define a few things before we get started.
I'll do my best to skim over the math so we can just get to the programming bits.

##### Grammar

A grammar is a set of rules that defines all the ways to structure a proper string of symbols in a language. These rules are called productions in parsing parlance. In natural written languages a grammar helps us write expressions of symbols that are 'generally' unambiguous and let us communicate meaning to other people that can read our language. A grammar is important because it's a way of determining if a string is valid in a particular language.

In programming it's nearly the same, we are trying to describe an algorithm in an exact way that the computer can understand. Most general purpose programming languages provide a flexible grammar that we can use to describe lots of different algorithms and if a language is [Turing complete](https://en.wikipedia.org/wiki/Turing_machine), we can describe *any* computable algorithm!

##### Terminal & Nonterminal Symbols

In a programming language some symbols are able to be broken down by productions into other types, these are called nonterminals. Nonterminals have rules that describe how they should be substituted. Some examples of nonterminals could be a function call, an expression, or an array of elements. Terminals on the other hand don't break down into any further parts, but don't worry we will see how to use both of these in the code. Some examples of a terminal might be a number, a variable name, or a character.

The idea is that you use the grammar productions to substitute the symbols in your string until you get down to terminals that you can run computations on. You can think of terminals as the data in your program, and the nonterminals as the operations you want to perform on the data.

##### Context-Free Grammar

We're not going to use just any grammar, we are using a context-free grammar. This means that the productions of the grammar are capable of determining if a string should be substituted as a nonterminal or not without the help of other contextual symbols around it. What this means in the math is that our productions must always be a single nonterminal that substitutes down to other terminals or nonterminals, but not the other way around.

An example of generating a string using a grammar is shown below. Nonterminals are uppercase and terminals are lowercase. Notice how we always go from a nonterminal on the left-hand side to another nonterminal or terminal on the right-hand side. That's the basis of a context-free grammar.


```
GRAMMAR RULES
    --- Left-Hand Non-Terminals
   |
   v
1: S' -> C <--- Right Hand Substitution
2: C -> X + Y
3: X -> a <--- Terminal Symbol
4: Y -> b

Example string to parse: a + b

Steps:
a + b   : Initial String
a + Y   : via Rule 4
X + Y   : via Rule 3
C       : via Rule 2
S'      : via Rule 1 - Finish with Start symbol
```

But how do we go about turning these rules into usable code? To do that we're going to use the [LALRPOP](https://github.com/nikomatsakis/lalrpop) library which will *generate* Rust code that can *parse* our grammar. That's why it's called a *parser generator*. For the time being LALRPOP implements an algorithm called [LR(1)](https://en.wikipedia.org/wiki/Canonical_LR_parser), also known as a Canonical LR parser. One caveat of using LR(1) is that our grammar must be unambiguous. **For any valid input string given to the parser it should only be able to parse the string one way.**

Without going into too many of the details, LR(1) uses **1** lookahead symbol when parsing. According to Wikipedia this allows for more flexible grammars. I'm just a language amateur so I can only assume whoever spends their free time editing language parsing wiki entries knows what they are talking about.

The 'LR' part of LR(1) means that the algorithm uses leftmost input and reverse rightmost derivation, which essentially means it reads in symbols 'L'eft-to-right and then applies grammar productions to the 'R'ightmost symbols first, working it's way back to the left most symbols. If you are looking at a parse tree, it's also called bottom up parsing.

[This Wikipedia section](https://en.wikipedia.org/wiki/Context-free_grammar#Derivations_and_syntax_trees) was one of the more easily digestible sections about left vs right derivations if you want to know more. [This article](https://en.wikipedia.org/wiki/LR_parser) also has a decent example of the LR parsing algorithm.

I still only have a cursory understanding of parsers, but luckily with LALRPOP you don't have to be a language expert to write parser code. My 'language' definitely won't be useful, but hopefully we'll learn a little bit.

#### THE CODE!

I'm making a few assumptions. 

1. You have Rust and Cargo installed on your computer. 
2. You understand enough Rust to read the code.

Luckily if you don't have those 2 things, there are some good resources [here](https://www.rust-lang.org/en-US/downloads.html) & [here](https://www.rust-lang.org/en-US/documentation.html) to help you on your journey.

##### The Setup

Lets first setup our new project with `cargo new --bin smellyalater`. Once we have that done we'll need to setup a build script for LALRPOP so it knows that when we compile with Cargo we want to actually generate code from our grammar. Here's what your `Cargo.toml` file should have

```toml
[package]
name = "smellyalater"
version = "0.1.0"
authors = ["Your Name <yourname@youremail.com>"]
# Our nice build script in our project root
build = "build.rs"

# Our nice dependencies
[build-dependencies]
lalrpop = "0.12.0"

[dependencies]
lalrpop-util = "0.12.0"
```

In our project root we also need to make a build script that we are calling `build.rs` that actually reads our grammar and generates the Rust code. Here's what `build.rs` looks like.

```rust
extern crate lalrpop;

fn main() {
    lalrpop::process_root().unwrap();
}
```

##### The Grammar Code

Finally we can start to write some code! Hooray! To start let's make a `src/smellyalater.lalrpop` file. This is where we will be writing our grammar. *It's handy to set your editor to use Rust syntax for this filetype, as the two are pretty close.*

```rust
use std::str::FromStr; // LALRPOP pulls in dependencies just like in Rust.

grammar; // This marks the file as a grammar for LALRPOP to generate code for.

pub Num: i32 = {
    <n:r"[-]?[0-9]+"> => i32::from_str(n).unwrap(),
};

```
Woah! What a load of code. Let's go over it. `pub Num: i32` is a nonterminal declaration. `pub` works the same as in Rust, publically exposing this type in our generated code. The `i32` says that the type of value we want this nonterminal to end up as in the generated code is an `i32`. 

Put your regex hardhat on. `<n:r"[-]?[0-9]+">` this part means when we are parsing we want to look for zero or more `-` characters and then any number of digits. Positive or negative integers essentially. The `n` in this case provides a binding to the value our regex matches against and lets us use that binding in the *action!* code. 

`=> i32::from_str(n).unwrap();` is our extreme *action!* code. It says, create an `i32` from a string, in this case `n` and return an unwrapped result, which in the case of a success will be an integer value.

Yay! We can build our project by using `cargo build` and we should get a hot-off-the-press parser in Rust. Our build script looks for all the `.lalrpop` files in our `src` directory and then generates a `src/<filename>.rs` for the `.lalrpop` file. In my case it will generate a `src/smellyalater.rs` file. 

We'll not look in that file just yet. It's a little scary in a *computer generated code* kind of way, but it does have order and you should skim through it eventually.

By now you might be thinking, we'll that's great I went through all this work but what can I do with this... *shrugs*. If you are an enterprising individual you could scour the landscape touting your new parser generating skills to the computationally hungry masses. In my case I just used it to write a crappy little REPL based on my custom grammar.

#### REPL REPL, Your face is a mess

I wanted a nice interactive way to test out my grammar (alternatively you can write actual tests for your grammar in `src/main.rs` and run `cargo test`). To do this I wrote a small REPL that uses that newly minted parser to read in strings and output results. As your grammar gets more complex this REPL will probably outlive it's usefullness, so your mileage may vary. Here's the code for the REPL in `src/main.rs`.

```rust
pub mod smellyalater; // synthesized by LALRPOP

use std::io::prelude::*;
use std::io;

fn main() {

    let mut input_line = String::new();

    loop {
        print!("> ");
        let _ = io::stdout().flush(); // Make sure the '>' prints

        // Read in a string from stdin
        io::stdin().read_line(&mut input_line).ok().expect("The read line failed O:");
        
        // If 'exit' break out of the loop.
        match input_line.trim() {
            "exit" => break, //This could interfere with some parsers, so be careful
            line => {
                // Use our parser 'library' to parse a num, we can use this
                // method because of the `pub` keyword
                println!("{:?}", smellyalater::parse_Num(&line.to_string()));
            }
        }
        
        // Clear the input line so we get fresh input
        input_line.clear();
    }

    println!("Smell ya later! ;D");
}
```

We can now run `cargo run`, go get a snack and by the time you get back we'll have a little REPL where we can turn a string into an `i32`!

Once again, that's all well and good, and if you are still here I'm sorry for the rollercoaster of emotions. Now that we have all of the pleasantries out of the way, we can get down to brass cats and build something with a little more bite.

#### Grammar School

For our little language we basically want something that can compute down into values immediately. Unfortunately in this example we won't go into dealing with memory management and function calls and virtual machines, as cool as that would be. Just like the LALRPOP tutorial I'm ripping off, we'll just be building a calculator. Let's get started.

##### Basic Design

The basic design for our grammar is layed out below.

```
GaRY likes ASh dislikes BrOcK hates MiSTY loves JESSiE and MEOwTH or Togepi
1011   +   110     -    10101   /   10111   *   111101  &  111011  | 100000
```

Each string in our input besides our keywords is a binary number and based on if the letter is upper case or lower case determines the value of the bit. Uppercase is `1` and lowercase is `0`, so a string like `AsH` is `101`. Then based on the keyword the two values on the ends of the keyword are operated on. So let's see how this looks as grammar productions.

```
S' -> <Expr>
Expr -> <Expr> and <Sum>
Expr -> <Expr> or <Sum>
Expr -> <Sum>
Sum -> <Sum> likes <Product>
Sum -> <Sum> dislikes <Product>
Sum -> <Product>
Product -> <Product> loves <Name>
Product -> <Product> hates <Name>
Product -> <Name>
Name -> i32
```

Notice a few things about this. First we are using many of the types recursively in most of the productions. This allows our language to be more flexible and we can nest expressions and chain together operations.

Also notice that our productions create a hierarchical order.

```
Expr -> Sum -> Product -> Name
```

This creates a precedence order, because the algorithm does reverse rightmost derivation a `Product` will get substituted before a `Sum` and a `Sum` before an `Expr`. Precedence is important because it tells the parser what gets parsed when and removes ambiguity from our grammar. The parse tree below shows how precedence would work with a rightmost derivation, also known as 'bottom-up' parsing, going from the bottom of the tree to the top.

```
Example Parse Tree: Brock likes mIsty loves asH

                    Expr
                     |
                    Sum
          ___________|___________
         /           |           \
        Sum       'likes'     Product
         |                       |
       Product          _________|________             
         |             /         |        \
        Name        Product   'loves'    Name
         |             |                   |
      'Brock'        Name                'asH'
                       |
                    'mIsty'
```

Let's see how to build this in LARLPOP and Rust.

##### Implementation

First we're going to need to setup another file. This one is `src/util.rs`. In it we'll add the code to convert the string to an i32.

```rust
pub fn parse_name(name: &str) -> i32 {
    let mut sum = 0;
    // Iterate over the string and sum up the binary values based on index
    for (index, c) in name.chars().rev().enumerate() {
        if c.is_uppercase() {
            sum += 2i32.pow(index as u32);
        }
    }
    sum
}
```

We'll also need to change a few other files. Our `src/main.rs` will need to add our new `util` module and fix which public function we are calling from the parser code. Like this

```rust
pub mod smellyalater; //synthesized by LALRPOP
pub mod util; // <--- Our new addition

use std::io::prelude::*;

// ... Previous code here

match input_line.trim() {
    "exit" => break,
    line => {
        // !!!!!! Here we changed parse_Num to parse_Expr to match our
        // grammar code
        println!("{:?}", smellyalater::parse_Expr(&line.to_string()));
    }
}

// ... More code under here

```

Finally we can see what our final `src/smellyalater.lalrpop` file looks like.

```rust
// use std::str::FromStr;
use util;

grammar;

pub Expr: i32 = {
    <n:Expr> "and" <e:Sum> => n & e,
    <n:Expr> "or" <e:Sum> => n | e,
    Sum,
};

Sum: i32 = {
    <s:Sum> "likes" <p:Product> => s + p,
    <s:Sum> "dislikes" <p:Product> => s - p,
    Product,
};

Product: i32 = {
    <p:Product> "loves" <n:Name> => p * n,
    <p:Product> "hates" <n:Name> => p / n,
    Name,
};

Name: i32 = {
    <name:r"[A-z]+"> => util::parse_name(name),
};

// Num: i32 = {
//     <n:r"[-]?[0-9]+"> => i32::from_str(n).unwrap(),
// };
```

As you can see our grammar looks pretty similar to the rules with only a few additions. Our action code has all of the operations for the `i32` values that we are returning, and our `Name` definition has a regex to find all the string values that are considered `Name`s in the `Expr` rules and send them into the `parse_name` function we wrote.

We can run `cargo run` again in our project root and if everything is correct we should get a REPL with a binary name calculator! Hooray! 😉

#### What's Next (Troubles & Additions)

This tutorial barely scratched the surface of LALRPOP. It has lots of features that I didn't cover. Some of the more important ones that you can read about in the original tutorial were macros so you don't have repeat a bunch of code in your definitions, and some more syntax features so you can handle things like Vectors of values and dealing with commas.

To add more to this tutorial people might want to add a an AST which is shown in the original tutorial. Adding some memory functionality so you can keep function names and add function calls. Playing around with the various grammar rules to build different types of languages would be interesting. Also remember that you can return any Rust type so you can build up complex datatypes from your parser, not just `i32`s. It would also be fairly easy to parse an input file with your expressions in it, and build a simple interpreter.

Most of the problems I had were finding good beginner resources on learning to understand the basics of language parsing, the algorithms, and the techniques that go along with it. There is a fair amount of terminology and notion to absorb and I'm still not super clear on the details, although now I have a starting point. 

Getting LALRPOP up and running is pretty easy given the tutorial, and once you start to understand the idea of unambiguity in your grammar things become a lot easier. LALRPOP also has nice errors that will give you a visual description of what the grammar and algorithm are doing. Niko Matsakis' [blog post](http://smallcultfollowing.com/babysteps/blog/2016/03/02/nice-errors-in-lalrpop/) goes into more details about error handling and parsing issues that arise.

I hope you learned a little bit, and this gave you at least a peek behind the curtain of how some programming languages and parser generators work, and maybe encouraged you to write your own language. I know this post got a little long but thanks for reading! 
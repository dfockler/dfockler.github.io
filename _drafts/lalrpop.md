---
layout: post
title: Let's Build a REPL with Rust & LALRPOP
tag: lalrpop
---

A lot of this post is kind of a pure ripoff of the [LALRPOP Tutorial](https://github.com/nikomatsakis/lalrpop/blob/master/doc/tutorial.md) which was written by an actual professional in language parsing. This is sort of my layman's interpretation of some of the concepts and ideas, so I can learn and share hopefully correctish insight and maybe get some other people to try stuff out that might seem scary at first.

#### A Highspeed Flyby of Programming Language Parsing

So first things first, we have to define a few things before we get started.
I'll do my best to skim over the math so we can just get to the programming bits.

##### Grammar

A grammar is a set of rules that defines all the ways to structure a proper string of symbols in a language. In written languages a grammar helps us write expressions of symbols that are 'generally' unambiguous and let us communicate meaning to other people that can read our language. In programming it's exactly the same, we are trying to describe an algorithm in an exact way that the computer can understand. Most general purpose programming languages provide a flexible grammar that we can use to describe lots of different algorithms and if a language is [Turing complete](https://en.wikipedia.org/wiki/Turing_machine), any computable algorithm.

##### Terminal & Nonterminal Symbols

In a programming language some symbols are able to be broken down by the grammar rules into component parts, these are called nonterminals. Nonterminals have rules that describe how they should be broken down and substituted into their constituent parts. Some examples of nonterminals could be a function call, an expression, or an array. Terminals on the other hand don't break down into any further parts, don't worry we will see how to use both of these in the code. Some examples of a terminal might be a number, a variable name, or a character. The idea is that you use the grammar rules to substitute the symbols in your string until you get down to terminals that you can run computations on. You can think of terminals as the data in your program, and the nonterminals as the operations you want to run on the data.

##### Context-Free Grammar

We're not going to use just any grammar, we are using a context-free grammar. This means that the rules of the grammar are capable of describing all possible strings in our language without the need for context to tell us if the string is valid, hence the context-free part. What this means in the math is that our rules must always be a single nonterminal symbol that substitutes down to yield other terminals or nonterminals, but not the other way around. We want to end up with terminals that we know how to run computations on.

An example of generating a string is shown below. Capital letters are nonterminal and terminals are lowercase. Notice how we always go from a nonterminal on the left-hand side to another nonterminal or terminal on the right-hand side. That's the basis of a context-free grammar.


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

But how do we go about turning these rules into code? To do that we're going to use the [LALRPOP](https://github.com/nikomatsakis/lalrpop) library which will *generate* Rust code that can *parse* our grammar. That's why it's called a *parser generator*. For the time being LALRPOP implements an algorithm called [LR(1)](https://en.wikipedia.org/wiki/Canonical_LR_parser), also known as a Canonical LR parser. One caveat of using LR(1) is that our grammar must be unambiguous. **For any valid input string given to the parser it should only be able to parse the string one way.**

Without going into too many of the details, LR(1) uses **1** lookahead symbol when parsing. According to Wikipedia this allows for more flexible grammars. I'm just a language amateur so I can only assume whoever spends their free time editing language parsing wiki entries knows what they are talking about. The 'LR' part of LR(1) means that the algorithm uses leftmost input and reverse rightmost derivation, which essentially means it reads in symbols 'L'eft-to-right and then applies grammar rule substitutions to the 'R'ightmost symbols first, working it's way back to the left-hand side. [This Wikipedia section](https://en.wikipedia.org/wiki/Context-free_grammar#Derivations_and_syntax_trees) was one of the more easily digestible sections about left vs right derivations if you want to know more. [This article](https://en.wikipedia.org/wiki/LR_parser) also has a decent example of the LR parsing algorithm.

I still only have a cursory understanding of parsers, but luckily with LALRPOP you don't have to be a language expert to write parser code. My 'language' definitely won't be useful, but hopefully we will learn a little bit.

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
use std::str::FromStr; // Pulls in dependencies just like in Rust.

grammar; // This marks the file as a grammar for LALRPOP to generate code for.

pub Num: i32 = {
    <n:r"[-]?[0-9]+"> => i32::from_str(n).unwrap();  
};

```
Woah! What a load of code. Let's go over it. `pub Num: i32` is a nonterminal declaration. `pub` works the same as in Rust, publically exposing this type in our generated code. The `i32` says that the type of value we want this nonterminal to end up as in the generated code is an `i32`. 

Put your regex hardhat on. `<n:r"[-]?[0-9]+">` this part means when we are parsing we want to look for zero or more `-` characters and then any number of digits. Positive or negative integers essentially. The `n` in this case provides a binding to the value our regex matches against and lets us use that binding in the *action!* code. 

`=> i32::from_str(n).unwrap();` is our extreme *action!* code. It says, create an i32 from a string, in this case `n` and return an unwrapped result, which in the case of a success will be an integer value.

Yay! We can build our project by using `cargo build` and we should get a hot-off-the-press parser in Rust. Our build script looks for all the `.lalrpop` files in our `src` directory and then generates a `src/<filename>.rs` for the `.lalrpop` file. In my case it will generate a `src/smellyalater.rs` file. 

We'll not look in that file just yet. It's a little scary in a *computer generated code* kind of way, but it does have order and you should skim through it eventually.

By now you might be thinking, we'll that's great I went through all this work but what can I do with this... *shrugs*. If you are an enterprising individual you could scour the landscape touting your new parser generating skills to the computationally hungry masses. In my case I just used it to write a crappy little interpreter based on my custom grammar.

#### REPL REPL, your face is a mess

I wanted a nice interactive way to test out my grammar. To do this I wrote a small REPL that uses that newly minted parser to read in strings and output results. As your grammar gets more complex this REPL will probably outlive it's usefullness, so your mileage may vary. Here's the code for the REPL in `src/main.rs`.

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

        input_line.clear();
    }

    println!("Smell ya later! ;D");
}
```

We can now run `cargo run` and we'll get a little REPL where we can turn a string into an `i32`!

Once again, that's all well and good, and if you are still here I'm sorry for the rollercoaster of emotions. Now that we have all of the pleasantries out of the way, we can get down to brass cats and build something with a little more bite.

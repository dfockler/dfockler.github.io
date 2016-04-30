---
layout: post
title: What the Heck are Lifetimes?
tag: rust-lifetimes
---
I'm not very familiar with the concepts of lifetimes. They are an important concept in the [Rust language](http://www.rust-lang.org/), so I'm gonna try and explain it while I learn! I also need to finally get code formatting setup on this blog. So let's begin!

To begin with, most of the stuff I'm talking about can be found over in the [Rust book](http://doc.rust-lang.org/nightly/book/). In the Rust book, lifetimes are not under their own section, but live in the Ownership section. Ownership is the way Rust manages to have memory safety without having a garbage collector. So how does a lifetime, which I haven't even explained yet, fit into all of this? Well, let's look at some code!

{% highlight ruby %}
for "HELLO WORLD!".each do
    puts "my banana is yellow"
end
{% endhighlight %}

That was a nice code block thanks to markdown
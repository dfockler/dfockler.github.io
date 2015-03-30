---
layout: post
title: Why not an Object?
tag: not-an-object
---
As I've been programming and reading more code and understanding it, I've come across an interesting OOP anti-pattern.
Many intermediate programmers generally have a decent grasp on the core syntax, standard libraries and other working parts of a language.  

What they don't yet have is a grasp on how to use OOP to it's full potential in their codebase. I've fallen into this trap, and I've seen it in some other people's code: using a deeply nested array or hash of values and passing it throughout their program while performing functions on the data inside of it.

If you find yourself passing a deeply nested array around in your code while performing manipulations on it's data, it's likely that you might want to turn this into an object or multiple objects. Often times it will start with a JSON object that is only going to be used in one section of the code, then a few places, then a lot of places. At that point it would make the code more readable and easy to change if you turned that gnarly array into it's own class. If you find that you are performing lots of operations on the data inside an array or hash, then it makes sense to refactor that into an object, with it's own methods and properties.

Structuring code using OOP in practice is actually pretty hard. In programming courses we only see things like cars inheriting from automobiles, and cats being polymorphically accessed as animals. This is great for learning but the tricky part comes when you are building a system of objects together and finding a way to make them flexible, testable, and extensible. So the next time you aren't really sure what's in that four dimensional array, maybe think about what might work better instead.
---
title: "Attaching event listeners in FOR loop"
publishDate: "10 Jan 2023"
description: "This article describes a bug with scoping in javascript for loop"
tags: ["javascript", "scope"]
---

## Introduction

A friend who was getting into tech was developing a GUI calculator in vanilla JS. In the process, he noticed a bug, and after searching the web, he found an article that talks about his problem [event listeners in loops](https://gomakethings.com/why-you-shouldnt-attach-event-listeners-in-a-for-loop-with-vanilla-javascript/)

This post is an in-depth explanation of the problem and possible solutions.

## Statement of Problem

In building a calculator application, there are a series of buttons for selecting particular numbers. To achieve the dry (do not repeat) principle and easier readability, it makes sense to loop through them and add an event listener that listens to the particular button that was clicked and uses it to perform other operations.

The code below shows an example of his first implementation.

```html
<!DOCTYPE html>
<html>
    <title>
        This is a test page
    </title>
    <body>
        <button class="btn">0</button>
        <button class="btn">1</button>
        <button class="btn">2</button>
        <button class="btn">3</button>
        <button class="btn">4</button>
        <button class="btn">5</button>
    </body>
    <script>
        let buttons = document.querySelectorAll('.btn');
        for (var i = 0; i < buttons.length; i++){
            buttons[i].addEventListener('click', function (){
                console.log(i)
            })
        }
    </script>
</html>
```
When any of the buttons is clicked, it logs the number `5`.

## Solution

The main issue here is scope. Since the event listener is not executed immediately, the value of `i` remains the same for all the buttons.
The reason is that hoisting occurs, and the declaration for the `var i` moves to the global scope. Therefore, when the event listener is triggered, all the buttons access the same `i` which is the last value for the loop. 

A code solution.

```html
<!DOCTYPE html>
<html>
    <title>
        This is a test page
    </title>
    <body>
        <button class="btn">0</button>
        <button class="btn">1</button>
        <button class="btn">2</button>
        <button class="btn">3</button>
        <button class="btn">4</button>
        <button class="btn">5</button>
    </body>
    <script>
        let buttons = document.querySelectorAll('.btn');
        for (var i = 0; i<buttons.length; i++){
            (function(){
                var number = i
                buttons[i].addEventListener('click',function (){
                    console.log(number)
                })
            }())
        }
    </script>
</html>
```
The declaration of `(function(){}())` scopes the number to that particular function object, and whenever the event listener is triggered, it references its own instance of number.

In fact, in Javascript, a way to create a namespace or keep declarations and variables in a place is to use this.

## Modern Solution

A more modern solution is to use the `let` keyword for the definition, which provides a type of scope.

```html
<!DOCTYPE html>
<html>
    <title>
        This is a test page
    </title>
    <body>
        <button class="btn">0</button>
        <button class="btn">1</button>
        <button class="btn">2</button>
        <button class="btn">3</button>
        <button class="btn">4</button>
        <button class="btn">5</button>
    </body>
    <script>
        let buttons = document.querySelectorAll('.btn');
        for (let i = 0; i < buttons.length; i++){
            buttons[i].addEventListener('click',function (){
                console.log(i)
            })
        }
    </script>
</html>
```
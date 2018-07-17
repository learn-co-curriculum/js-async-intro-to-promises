# Coding for Asynchrony with `Promise`s

## Problem Statement

In an asynchronous execution model, like JavaScript uses in the browser,
handling asynchronous work, responding to success, and responding to failure
could create a lot of twisty, hard-to-read code.

However, ES2015 introduced `Promise`s, which allowed us to code for
uncertainty _expressively_. In this lesson we'll see how `Promise`s work and
how they secretly make `fetch()` so clean.

## Objectives

1. Demonstrate pre-`fetch()` data fetch code
2. Demonstrate capturing the `Promise` returned by `fetch()`
3. The hidden `Promise`s of `fetch()`
4. Handling failure, or `rejected` `Promise`s

## Demonstrate Pre-`fetch()` Data Fetch Code

Formerly, if you wanted to retrieve data asynchronously, here's the code you
would write.

```js
//set up the request
let xhr = new XMLHttpRequest();
xhr.open('GET', 'http://api.open-notify.org/astros.json');
xhr.responseType = 'json';

//provide a function to call if the request is successful
xhr.addEventListener("load", () => console.log(xhr.response))

//provide a function to call if the request is successful
xhr.addEventListener("error", () => {
  console.error(xhr.response);
  // Do cleanup code...
})

//send the request
xhr.send();
```

There are a number of weaknesses here that really bothered JavaScript
developers. Let's list two obvious pain points:

* The amount of setup for each data source is pretty big. It was pretty common
  to find developers writing functions that abstracted this setup "boilerplate"
  code:
  `let ajaxFactory = (source, successFn, failFn) => { let xhr = new XMLHttpRequest(); ...; return xhr }`
* If you had an XHR request that depended on the success of another XHR,
  the code got really nasty between nested callbacks. You could imagine some
  web-kiosk code that said "Ask the US Government's atomic clock for the day 
  of the week (Request 1) AND based on the day of the week, request that day's
  menu for lunch to put on the cafeteria's monitor." What happens if one of those
  requests time out? How do you _expressively_ write code that's not a huge knot
  to read / debug / maintain? It's hard
  
Taking inspiration from the languages of Lisp and Prolog, JavaScript developers
wrote their own `Promise`-like implementations in libraries like jQuery, Bluebird,
and `Deferred.js`. It was clear that the community needed a better, standard, way
to capture and work with code that operated in an uncertain world.

## Demonstrate Capturing the Promise Returned by `fetch()`

Use a DevTools console inside an incognito window to see the following:

```js
let fetchPromise = fetch("http://api.open-notify.org/astros.json")
fetchPromise.constructor // => ƒ Promise() { [native code] }
fetchPromise.then // => ƒ then() { [native code] }
```

The `constructor` property tells us that `fetchPromise` is a `Promise` and that
it has a method called `then` on it. We can see that this agrees with the
`Promise` [documentation][promdoc] at MDN.

When a `Promise` is _fulfilled_, i.e. it successfully does something that might
fail, then function inside the `then()` is called. The return value of _that_
function is wrapped in a _new_ Promise so that, if so desired, _another_
`then()` can be attached to it.

If a `Promise` is _rejected_, i.e. the thing did not happen, then the function
inside the nearest `catch()`, and all subsequent `catches()` are executed.

Let's see how this works with our `fetch()` skeleton assuming that all of the
`Promise`s _resolve_ i.e. are successful.

## The Hidden `Promise`s of `fetch()`

The `fetch()` method uses `Promise`s, because some asynchronous operations are
risky.

```js
fetch("URL") // Point 1
  .then(resp => resp.json()) // Point 2
  .then(json => console.log(json)) // Point 3
```

At Point 1, we're making a standard synchronous call. Nothing new. But that
call returns a Promise (Point 2). Why does it do that? Well, what if the
network fails midway? You can't say "Hey response, give me your JSON" if a
response never came back. Or what if someone deployed bad code to the server?
If the server says "Go away", we won't get a `.json()`-able response.

Assuming that the `fetch()` at Point 1 is _fulfilled_ the `then()` in Point 2
executes and the return value is wrapped in a Promise and sent to the `then()`
at Point 3.

But wait, why does Point 2 return a `Promise`? What could go wrong with asking
a _fulfilled_ response for its JSON? What if the server returns a garbage
string that's _not_ JSON? Again, **if** that conversion can be done it is
wrapped in a `Promise` and is available to the `then()` in Point 3. Inside the
callback at Point 3 we know we have valid JSON that we can proceed to use.

## Handling Failure, or `rejected` `Promise`s

Let's try this code in the DevTools console inside an incognito window:

```js
fetch("http://api.open-notify.org/astros.json")
  .then( resp => console.log("Yay"))
  .catch( error => console.error(`Oh no! ${error}`));
```

This should work. But let's make an error in the URL.

```js
fetch("http://api.open-notify.zrg/astros.json")
  .then( resp => console.log("Yay"))
  .catch( error => console.error(`Oh no! ${error}`));
```

This returns "Oh no! Failed to fetch" Obviously, there's no such thing as
`api.open-notify.zrg`.

Earlier we said that in an event of failure, all the callbacks in all the
`catch()` calls will be executed. Let's see how that works.

```js
fetch("http://api.open-notify.org/astros.json")
  .then( resp => resp.json())
  .catch( error => console.error(`Oh no! ${error}`))
.then( json => console.log(json["number"]))
.catch( error => console.error(`Ruh-roh! Couldn't convert the json: ${error}`))
```

This works. Let's break the URL again:

```js
fetch("http://api.open-notify.zrg/astros.json")
  .then( resp => resp.json())
  .catch( error => console.error(`Oh no! ${error}`))
.then( json => console.log(json["number"]))
.catch( error => console.error(`Ruh-roh! Couldn't convert the json: ${error}`))
```

In this case we get two errors:

```text
Oh no! TypeError: Failed to fetch
Ruh-roh! Couldn't convert the json: TypeError: Cannot read property 'number' of undefined
```

This makes sense. Because the first `Promise` failed, all other `Promise`s that
were contingent on its success _also_ need to fail. We humans understand the
risk associated with building our future on too many what-ifs:

> If I make it in Hollywood, I will get a big house or else I will move back
> home; and if I have a big house in that big house I will have a pony, but if
> not I will not need a stable-hand.

`Promise`s allow you to express to yourself and other developers your awareness
that success is by no means assured, even when **you** do everything correctly.

## Conclusion

`Promise`s allow JavaScript developers to expressively communicate the risk
inherent in certain type of work. They are part of the internals of the
`fetch()` method used to retrieve data when using the AJAX technique.

## Resources

* [HTML 5 Rocks on `Promise`][h5r]

[h5r]: http://www.html5rocks.com/en/tutorials/es6/promises/

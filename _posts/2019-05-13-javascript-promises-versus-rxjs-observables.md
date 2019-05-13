---
layout: post
title: 'JavaScript Promises Versus RxJS Observables'
date: 2019-05-13 09:00:00 +0700
categories: [javascript]
tags: [Promises, RxJS, Observables]
image: place_holder.jpeg
---

With the formal introduction of Promises in the ES2015 version of JavaScript they have become the primary way to handle async code. Promises are a fantastic way to handle async code in a composable maintainable way. Sometimes in more complex situations Promises can fall short. In this post, we are going to compare and contrast the JavaScript Promise API to the Observable library RxJS. We will see how similar Promises and Observables are as well as how they differ and why we would want to use Observables over promises in certain situations.

## JavaScript Promises

Before we jump into Observables, let’s do a quick review of the Promise API. For our example, we are going to create a short set timeout to simulate some kind of async task for our Promise to handle.

```javascript
const promise = new Promise(resolve => {
  setTimeout(() => {
    resolve('Hello from a Promise!');
  }, 2000);
});

promise.then(value => console.log(value));
```

Our Promise is instantiated with the _new_ keyword. In the constructor, we pass in a function that will pass a reference to a _resolve_ function. Using this special resolver function, we can run some async task and when it completes passing the value to the _resolve_ for the consumer of our promise to receive the value.

This is a pretty standard way of handling async code in JavaScript applications. There are some issues though that can arise with more complex applications. For one what happens if we want multiple async values or events, say from a WebSocket? Well, this is where Observables can help out.

## RxJS Observables

An Observable is a unique Object similar to a Promise that can help manage async code. Observables are not part of the JavaScript language yet but are being proposed to be [added to the language](https://github.com/tc39/proposal-observable). Since we can’t use a built-in Observable, we rely on a popular Observable library called [RxJS](http://reactivex.io/). RxJS provides an Observable implementation for use to use as many other helpful utilities related to Observables.

First, let’s go ahead and look at a simple example of an Observable. This example we demonstrate the same functionality as our Promise example we saw earlier.

```javascript
import { Observable } from 'rxjs';

const observable = new Observable(observer => {
  setTimeout(() => {
    observer.next('Hello from a Observable!');
  }, 2000);
});

observable.subscribe(value => console.log(value));
```

In this example, we import the _Observable_ from _rxjs_. Just like a Promise we instantiate and create our Observable by calling the new keyword. In the constructor, we pass a function that will handle our async task. The Observable will pass us a reference to an object called an Observer. The Observer is similar to the resolve function from our Promise example.

In the Observable, we create a _setTimeout_ like our Promise example. In the Observable we call _observer.next()_ to trigger and emit our value to the consumer of our Observable. To get the value of our Observable we call the _subscribe()_ method.

```javascript
observable.subscribe(value => console.log(value));
```

The _subscribe()_ method similar to the Promise _then()_ will pass the value to our function when the async task has completed. As you can see the Observable API is very similar to the Promise API. Observable have the same feature set as Promises plus additional features we will cover next.

## Multi Value Observables

We have seen how Observables are very similar to Promises, but what is different between the two? One of the significant differences between Observables and Promises is Observables support the ability to emit multiple asynchronous values. A Promise once it has resolved its async value it completes and can no longer be used. The one shot use falls short for the use case where we need multiple values over time. Some common use cases of this, web sockets with push notifications, user input changes, repeating intervals, etc.

Our next example is going to show how to create an Observable just like our previous example but instead of using a _setTimeout_ we will use a _setInterval_ to show multiple values.

```javascript
import { Observable } from 'rxjs';

const interval = new Observable(observer => {
  let count = 0;
  const interval = setInterval(() => {
    observer.next(count++);
  }, 1000);

  // once we stop listening to values clear the interval
  return () => {
    clearInterval(interval);
  };
});

interval.subscribe(value => console.log(value));
// ----1----2----3---->
```

In this example, we have a new Observable with a _setInterval_. Notice we still call _observer.next()_ to emit our value. With Observables, you can call _emit()_ multiple times yielding multiple values. Multi-value support is the big advantage to Observables over Promises. Now since we can have long-running async tasks in our Observable like a _setInterval_ we need to stop any tasks once we don’t care about receiving any more values. This leads us into unsubscribing from Observables.

With Observables, we can cancel them or unsubscribe from them when we no longer care about the values. To do this let’s look at the following code.

```javscript
const subscription = interval.subscribe(value => console.log(value));
setTimeout(() => subscription.unsubscribe(), 3000);
// ----1----2----3---->
```

When we subscribe to an Observable, we get back a subscription Object. With the Subscription object we can call _unsubscribe()_ at a later point in time. In this case, we subscribed to the Observable and got three values in three seconds. After three seconds we unsubscribe and stop receiving values. The next advantage of using RxJS is we get many utility functions called Operators to make it easy to deal with these streams of events.

## RxJS Operators

Operators are special utility functions similar to array functions make it easy to transform values before subscribing to them. Let’s look at our first Operator, the _map_ Operator.

```javascript
import { map } from 'rxjs/operators';

interval
  .pipe(map(value => value * value))
  .subscribe(value => console.log(value));

// ----1----2----3----4---->
//      map => x * x
// ----1----4----9----16--->
```

Using the same interval Observable we had before we call a method called _.pipe_. The _pipe_ method takes one to many Operator functions. This example we are using _map_ operator. The _map_ operator is similar to the JavaScript array map function. With the map operator we can map over or loop over each event as it is emitted. In this example, we take the emitted value and square the value. Now the receiving consumer of our Observable gets the squared value.

Another common RxJS operator is the _filter_ operator. The _filter_ operator is similar to the JavaScript filter array function. Using the _filter_ operator, we can filter out only specific values we care about.

```javascript
import { map } from 'rxjs/operators';

interval
  .pipe(
    map(value => value * value),
    filter(value => value % 2 === 0),
  )
  .subscribe(value => console.log(value));

// ----1----2----3----4---->
//      map => x * x
// ----1----4----9----16--->
//          filter
// ---------4---------16--->
```

We can chain the operators together. In our example, we filter our odd numbers in our Observable stream. A use case for this could be a push notification system where the user can filter out only the notifications they care about.

Hopefully, you have learned something new and have a better understanding of how Observables work compared to Promises. If you want to learn more about Observables and a little bit of Angular, check out this [NgHouston Meetup](https://coryrylan.com/blog/reactive-programming-with-rxjs-and-angular-ng-houston). The working code examples can be found below.

Source: [JavaScript Promises Versus RxJS Observables](https://coryrylan.com/blog/javascript-promises-versus-rxjs-observables)

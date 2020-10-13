---
title: Scheduling in React 16.x
date: 2020-06-24 16:00:00 +07:00
tags: [javascript, react, react fiber]
description: ....
image: "/assets/img/scheduling-in-react-16-x/background.jpeg"
---

# What is scheduler?

Let’s start with some theory.

JavaScript is a single-threaded language. This means we have one call stack which can perform one piece of code at the time. Among executing your code, the browser needs to perform a different kind of work. This includes:

- managing events (user clicks handlers, _setTimeout_ callbacks, etc.)
- layout calculations (building DOM / CSSOM) and
- repaints (based on the layout calculations)

Let’s focus on the last one. Within one second, browsers usually repaint 60 times, which means there is a single repaint every 16.6ms (more or less, depending on the environment).

When you run a piece of synchronous code for about one second, you will accidentally drop ~60 frames which introduces delays, hits UX, and makes your app unresponsive.

So, the primary objective of the scheduler is to balance between all the activities being performed within the browser, not starve any of them as well as keeps your app repaint-aware.

But, as developers, should we care? No doubt, we should be aware of the problem. Nowadays, we outsource quite a lot of responsibilities to third party entities while developing our apps.

Web frameworks do a lot for us, they manage routing and state, provide change-detection mechanisms, two-way-data-binding (Angular/Vue.js), update DOM directly, and many, many more. They act as a middleware between us and the browser, so it seems to be a great place for scheduling problem to be solved, right?

And web frameworks do so. There is no exception for React as well (at least, React 16.x).

# Trying to implement a simple scheduler

I will explain how it is managed by React, but, to understand the underlying problem, let’s write some simple piece of code…

Let’s simulate a heavy browser app:

```js
setInterval(() => {
    document.body.appendChild(document.createTextNode('hi'))
}, 3)
```

It forces the browser to do a repaint (caused by DOM manipulations being made each 3 ms). Now, consider this code:

```js
setInterval(() => {
    document.body.appendChild(document.createTextNode('hi'))
}, 3)
render()
function render() {
    for (let i = 0; i < 20; i++) {
        performUnitOfWork()
    }
}
function performUnitOfWork() {
    sleep(5)
}
function sleep(milliseconds) {
    // sleep for given {milliseconds} period (synchronous)
}
```

What is the impact of our browser rendering mechanism? Well, _render_ is synchronous, so it freezes browser for 100ms (20 * 5ms = 100ms). No repaint can be made within the time. User input events callbacks neither. Let’s take a look into Chrome’s "Performance tool" output for a given code:

<figure>
<img src="/assets/img/scheduling-in-react-16-x/performance-1.png" alt="fence">
<figcaption>Fig 2. Performance listing (ordinary implementation)</figcaption>
</figure>

In the example, we drop frames for more than 100ms (see vertical dotted lines, they show you when the frame ends). As mentioned earlier — the browser does a repaint each ~16.6 ms (60 frames per second). In this case, repaints are delayed which freezes any animation performing at the time. User input handlers are delayed as well. This is not what we want, right?

Now, let’s assume the provided _render_ function is a React 15.x implementation of render mechanism. During the process, React tracks changes, calls our life-cycle methods, compares props, etc. It is a time-consuming process that might take a long time to compute, especially in heavy apps.

Does the React 15.x have a scheduler mechanism at all? Nope, it does not, so there is no difference between our synchronous _render_ function and React 15.x _render_ implementation.

<figure>
<img src="/assets/img/scheduling-in-react-16-x/react-stack.png" alt="fence">
<figcaption>Fig 3. App structure</figcaption>
</figure>

See how it works [here](https://claudiopro.github.io/react-fiber-vs-stack-demo/stack.html). Quiet laggy, right? Let’s go back to our example… How can we improve this synchronous-based rendering mechanism? How about adding _setTimeout_?

```js
...
function render() {
    performUnitOfWork()
    setTimeout(render, 0)
}
...
```

What are the results?

<figure>
<img src="/assets/img/scheduling-in-react-16-x/performance-2.png" alt="fence">
<figcaption>Fig 4. Performance listing (with setTimeout)</figcaption>
</figure>

Yep, way better, our frames are no longer delayed. Our `performUnitOfWork` function is being handled separately, on each frame. Repaints are being made regularly — with no frame delay.

So, is it the way the scheduling problem is being solved for React 16.x apps? Not really. Among entire internal engine rewrite (so-called React Fiber) React team has introduced a dedicated module ("React Scheduler") to address scheduling issues. How it works? I will explain it soon, but firstly — we need to introduce... [Channel Messaging API](https://developer.mozilla.org/en-US/docs/Web/API/Channel_Messaging_API).

# MessageChannel-based scheduling

What it does is a simple thing. It lets you communicate across different JavaScript contexts, i.e. between your code and iframe, or between your code a web worker’s context.

Consider the following example:

```js
// index.html
<iframe src="iframe-page.html"></iframe>
<script>
var iframe = document.querySelector('iframe')
var channel = new MessageChannel()
iframe.addEventListener('load', () => {
    channel.port1.onmessage = e => console.log(e.data)
    iframe.contentWindow.postMessage('hi!', '*', [channel.port2])
})
</script>
// iframe-page.html
<script>
window.addEventListener('message', event => {
    console.log(event.data)
    event.ports[0].postMessage(
        'Message back from the IFrame'
    )
})
</script>
```

What you need to do is to set a listener on one port (`port1`), and transfer another (`port2`, which will be used by sender) into another context (i.e. to iframe) using `postMessage` API. Then you can communicate in both directions.

But do we need to communicate with iframe or WebWorker in our app? Not really, but the API (alongside communication capabilities) has another advantage. It lets you kindly schedule work that respects other browser activities, such as rendering, DOM calculation, etc.

How? By the so-called "messages loop" mechanism. Now, let me replace the previous implementation of synchronous _render_ function with this:

```js
setInterval(() => {
    document.body.appendChild(document.createTextNode('hi'))
}, 3)
const channel = new MessageChannel()
render()
function render() {
    channel.port1.onmessage = onMessageReceived
    channel.port2.postMessage(null)
}
function onMessageReceived(event) {
    performUnitOfWork()
    channel.port2.postMessage(null)
}
function performUnitOfWork() {
    sleep(5)
}
```

Our render function was updated to take advantage of MessageChannel API. We just created the "message loop". Sending a message on `port2` (inside `render` function) forces `onmessage` (which is `onMessageReceived`) to be called, which do some calculations (`performUnitOfWork`). Then once again it forces sending a message on port2, which eventually... yep... you see the pattern, right? That is how the message loop is created.

Now let’s look at our performance profile:


<figure>
<img src="/assets/img/scheduling-in-react-16-x/performance-3.png" alt="fence">
<figcaption>Fig 4. Performance listing (with message-loop)</figcaption>
</figure>

Repaints are being made each 14ms / 19ms. During the single frame, we can see `performUnitOfWork` is executed even 3 times which is better than _setTimeout_ approach. What is more, in the _setTimeout_ solution there were lots more empty spaces between "execution bars" where the browser does nothing. It’s not the case in the message loop.

# Scheduling in React 16.x

And here comes the fun part. I just described to you how does the most recent implementation of React Scheduler works. It uses the MessageGlobal API to achieve their in-browser scheduling goals. This approach is even better comparing to the _setTimeout_ implementation, as it is eventually able to perform even more work within the frame.

Now, this approach requires splitting a work into smaller chunks. Our example implementation of render function performs computation (`performUnitOfWork`) for about 5ms. And there is no difference for React implementation — it also splits work into smaller chunks to be executed within 5ms.

Why do we want to break our execution into smaller chunks? To let the main thread to execute pending events / do repaints / manage animation in the meantime, so there are no UX lags.

Consider this piece of React [code](https://github.com/facebook/react/blob/v16.13.1/packages/react-reconciler/src/ReactFiberWorkLoop.js#L1467-L1472):

<figure>
<img src="/assets/img/scheduling-in-react-16-x/work-loop-concurrent.png" alt="fence">
<figcaption>Fig 5. React's workLoopConcurrent function</figcaption>
</figure>


This is one of the most important fragment of React code. Main work loop. As mentioned earlier, the latest version of React (in the contrary to React 15.x) lets you split work into small chunks (`workInProgress`). Each of them is being handled (`performUnitOfWork`) one by one, as long as:

1. we have some work to do (`workInProgress !== null`)
2. and `shouldYield()` returns false.

So many synchronous work is being made within the `performUnitOfWork`. This piece of code comes from React’s "reconciler" mechanism. If you would like to find out more how React works under the hood, I strongly recommend you to search for "React Fiber" phrase. For the sake of the article, let’s assume `performUnitOfWork` is a function that walks our component tree, do change-detection computations, including call lifecycle methods, side-effects marking. It is a heart of React render process.

React Fiber is designed in the way, that each finished work result is being saved on the heap, so we can interrupt a `workLoopConcurrent` loop any time, and return to this later on.

But how do we know when to interrupt this work?

Here comes the `shouldYield` function which is a part of a "Scheduler" module. It has one responsibility — to decide whether to stop or continue working on tasks (`performUnitOfWork`). It returns `true` when we should stop computation and yield to the main thread, or `false` when we should proceed with our computation.

What is [an implementation of shouldYield](https://github.com/facebook/react/blob/v16.13.1/packages/scheduler/src/forks/SchedulerHostConfig.default.js#L164-L166)?

<figure>
<img src="/assets/img/scheduling-in-react-16-x/should-yield-to-host.png" alt="fence">
<figcaption>Fig 5. React's shouldYieldToHost function</figcaption>
</figure>



It checks whether we exceeded the deadline. What is the deadline? It is a currentTime + 5m, so it appears, React scheduler breaks the execution each
5 ms, so the same, as presented in our example earlier in the article. There is [a descriptive comment in the source code](https://github.com/facebook/react/blob/v16.13.1/packages/scheduler/src/forks/SchedulerHostConfig.default.js#L115-L119) presenting the way it works:


<figure>
<img src="/assets/img/scheduling-in-react-16-x/yield-interval.png" alt="fence">
<figcaption>Fig 5. React's yieldInterval constant</figcaption>
</figure>

Ok, where is a code presenting MessageChannel loop?

<figure>
<img src="/assets/img/scheduling-in-react-16-x/perform-work-until-deadline.png" alt="fence">
<figcaption>Fig 5. React's performWorkUntilDeadline function</figcaption>
</figure>

It’s a little bit more complicated, but at [the very bottom of the code listing](https://github.com/facebook/react/blob/v16.13.1/packages/scheduler/src/forks/SchedulerHostConfig.default.js#L189-L226), we have a channel setup. You can briefly walk through the code comments to see the flow. Inside `performWorkUntilDeadline` there is `port.postMessage(null);` which keeps the message loop running.

We have also a piece of code, which prepares deadline (`deadline = currentTime + yieldInterval;`)

What is `scheduledHostCallback`? Long story short, it eventually triggers `workLoopConcurrent` presented earlier, which is the main work loop for the React render process. But the function is designed to return information, whether work is done or we should continue in the next message loop iteration (`hasMoreWork`).

Remember React 15.x app example which presents [scheduling problem](https://claudiopro.github.io/react-fiber-vs-stack-demo/stack.html)?


<figure>
<img src="/assets/img/scheduling-in-react-16-x/react-stack.png" alt="fence">
<figcaption>Fig 3. App structure</figcaption>
</figure>

[Here](https://claudiopro.github.io/react-fiber-vs-stack-demo/fiber.html) you can see the same app but powered with React 16.x and its new React Fiber approach. Way better, right?

# React’s Scheduler Module

`shouldYield` is not the only API exposed to the internal use for React devs.

<figure>
<img src="/assets/img/scheduling-in-react-16-x/scheduler-api.png" alt="fence">
<figcaption>Fig 3. Scheduler API</figcaption>
</figure>

Considering the naming convention of [the exposed functions](https://github.com/facebook/react/blob/v16.13.1/packages/scheduler/src/Scheduler.js#L415-L434), we can conclude the module is still in development stage and API might change at the time.

Beware! This API is not intended to be used by the developers (like you and I). It is only for the team members/contributors internal usage. Of course, we use it, but indirectly.

You can see that the module exports function `runWithPriority` as well as some predefined constants (`UserBlockingPriority`, `NormalPriority`, `LowPriority`). It is not yet widely used in React, but the purpose of this is to let React schedule work with different priorities. We can expect more use cases to be supported with this API in further React releases.

What is the goal of such logic? To prioritize render of specific elements. In the Facebook app, it is more important to see news feed in the first place rather than header, footer or authenticated user section. In this case, we might render news feed related components with higher priority.

That’s the goal, hope to see some real use cases in upcoming React releases. It’s the future, but how about now? Where do we schedule a task using the Scheduler Module?

Well, hook’s effects [are being scheduled](https://github.com/facebook/react/blob/v16.13.1/packages/react-reconciler/src/ReactFiberWorkLoop.js#L2190-L2193) using React Scheduler Module.

There is an enigmatic `enqueuePendingPassiveHookEffectMount` [function](https://github.com/facebook/react/blob/v16.13.1/packages/react-reconciler/src/ReactFiberWorkLoop.js#L2182-L2196) which schedules `scheduleCallback` tasks with `NormalPriority`.

Let’s focus on the function name (`enqueuePendingPassiveHookEffectMount`) and find out what each name piece means:

- `enqueuePending`...: Scheduler module under the hood builds a list of callbacks (and assigned to it priorities) that needs to be called, so enqueue prefix is accurate in this context.
- ...`PassiveHookEffect`...: This is real fun. What are *passive hook effects*?:

We have two categories of effects, *passive* (_useEffect_) and *layout* (_useLayout_). Imagine this component:

```js
const App = () => {
    useEffect(() => {
        console.log('passive effect')
    }, [])
    useLayout(() => {
        console.log('layout effect')
    }, [])
}
```

Which of the effects will be called first? *layout effect*. Why? As mentioned in docs, layout effect is being called just *after DOM mutation*, but before *browser render*. *Passive effects*, on the other hand, are deferred (using scheduler module) to be called *after browser render*.

- _...Mount_: The easiest way to understand *mount*, as well as *unmount* hook effect is the example:

```js
useEffect(() => {
    // this is "mount" passive hook
    return () => {
        // this is "unmount" passive hook
    }
}, [])
```

Depending on the state of the component, React Reconciler module calls *mount* or *unmount* (i.e. while destroying component).

To sum it up all *passive effect* hooks are being called *asynchronously* — after browser repaint. This is a different approach comparing to the old implementation of _componentDidMount_ or _componentDidUpdate_ which block the browser from repainting. It is also worth noticing, that we have no alternative for _componentWillMount_ or componentWillUpdate in hooks world. Those methods used to be a great place to introduce side-effects by developers (which might again — delay repainting) so that they decide to deprecate it.

So, all the code you put inside *passive effect* goes through the React Scheduler.

# What is ‘isInputPending’?

This feature is a result of Facebook engineers’ struggles to shorten a time a user input (i.e. click, mouse, keyboard events) are being performed.

Let’s look at the example:

```js
while (workQueue.length > 0) {
    if (navigator.scheduling.isInputPending()) {
        // Stop doing work if we have to handle an input event.
        break;
    }
    let job = workQueue.shift();
    job.execute();
}
```

Via `navigator.scheduling.isInputPending()` we can find out whether there is a pending user event (click, mouse, keyboard, drag&drop, etc.) so we can quickly react by yielding execution to the main thread (i.e. via the `break` statement) to handle the event.

Of course, it’s an experiment, not the official standard. It was available at the Chrome browser from version 74 to 78 (until Dec 4, 2019) as a part of [Chrome Origin Trials](https://developers.chrome.com/origintrials/#/view_trial/4544132024016830465).

You can read more about it both on [Gihub](https://github.com/WICG/is-input-pending) as well as on [Facebook’s engineering blog](https://engineering.fb.com/developer-tools/isinputpending-api/).


But why do I describe this thing here, in scheduler-based article? Well, the interesting fact is, that [the current implementation](https://github.com/facebook/react/blob/v16.13.1/packages/scheduler/src/forks/SchedulerHostConfig.default.js#L127-L161) of React Scheduler takes advantage of the feature:


<figure>
<img src="/assets/img/scheduling-in-react-16-x/is-input-pending.png" alt="fence">
<figcaption>Fig 3. React's isInputPending fallback implementation</figcaption>
</figure>

After meeting specific requirements:

- our React app build has a feature flag enabled (`enableIsInputPending`)
- we use Google Chrome with experimental `navigator.scheduling.isInputPending` feature enabled

our React Scheduler module provides an improved version of `shouldYield` implementation, which yields execution to the main thread when there is any pending event waiting to be executed.

# History of scheduling in React

The problem with React 15.x was that the entire rendering process was being made synchronously. One, large, time-consuming, synchronous, and recursive (!) piece of code. This can not co-operate with other browser activities, right?

At the very beginning of React Fiber’s development (16.x), the React team was using `requestIdleCallback`. But, it appeared to be — as [Dan Abramov said](https://github.com/facebook/react/issues/11171#issuecomment-335431662) — not as aggressive as the team wanted to.

<figure>
<img src="/assets/img/scheduling-in-react-16-x/dan-said.png" alt="fence">
<figcaption>Fig 3. Dan about requestIdleCallback</figcaption>
</figure>

The next step was to simulate `requestIdleCallback` with `requestAnimationFrame`. They tried to guess frame size and align React render mechanism with vsync cycle. And it was ok, until the latest made (by Andrew Clark) which gets rid of `requestAnimationFrame`, as described [here](https://github.com/facebook/react/pull/16271)).

# Summary

As presented earlier, we have a couple of API which might be helpful, like `requestAnimationFrame`, `requestIdleCallback`, _message loop_ (MessageChannel), or even `setTimeout`.

But hey — there were designed for slightly different purposes, weren’t they?

No doubt, the scheduling problem is on the table, some of the concepts have been announced lately within the browser vendor environments. If you are interested in the topic — I recommend [WICG/main-thread-scheduling](https://github.com/WICG/main-thread-scheduling) repo as well as its "[Further Reading](https://github.com/WICG/main-thread-scheduling#further-reading--viewing)" section.


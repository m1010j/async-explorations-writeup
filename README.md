# Async recursion: promising, surprising, but foremost confusing

## I set out to write asynchronous recursive functions. I wasn't prepared for what I discovered.

JavaScript is a single-threaded language, but browsers use multiple threads to handle asynchronicity. I set out to explore how much we can leverage the multi-threaded nature of browsers with asynchronous recursive functions. The results were decidedly mixed and deeply confusing. And along the way I stumbled upon a number of surprising facts:

- In Chromium-based browsers, unlike in Firefox, Safari, and Edge, some asynchronous recursive functions are executed using multiple logical cores.
- But other, more computationally intensive, asynchronous recursive functions are executed using just one logical core in all browsers.
- Both synchronous and asynchronous recursive functions run more slowly when invoked by a Web Worker than when invoked in the main thread.
- On iOS, Chromium-based browsers never use multiple logical cores.
- On Android, Chromium-based browsers execute some asynchronous recursive functinos even faster than the number of logical cores available to them would suggest.
- Memoization doesn't have any apparent performance benefit for asynchronous recursive functions.

To obtain my results, I wrote [Explorations in Asynchronicity][explorations], a benchmark and analysis tool. Here I discuss the most surprising results. Readers are invited to follow along by exploring [more results][explorations-results], analyzing the raw data available through the [public API][api], or [running benchmarks][explorations-benchmarks] on their own devices.

### Fibonacci: synchronous and asynchronous

I used a regular Fibonacci function without memoization to establish a baseline:

```javascript
function syncFib(n) {
  if (n <= 0) return 0;
  if (n === 1) return 1;

  return syncFib(n - 1) + syncFib(n - 2);
}
```

This is a classic `O(n ^ 2)` function, as exhibited by the average calculation times of `syncFib(1)` through `syncFib(45)`:

![all browsers avg time sync result][all-avg-sync]

I started by comparing `syncFib` with an asynchronous version:

```javascript
async function asyncFib(n) {
  if (n <= 0) return 0;
  if (n === 1) return 1;

  const prevValues = await Promise.all([asyncFib(n - 1), asyncFib(n - 2)]);

  return prevValues[0] + prevValues[1];
}
```

`asyncFib` is an async function that uses `Promise.all` to simultaneiously make the recursive calls. It then awaits until both of those calls are resolved and returns their sum.

How would browsers handle this? Would they use multiple threads to process `asyncFib(n - 1)` and `asyncFib(n - 2)` in parallel? Or would the latter have to wait for the completion of the former?

To my initial surprise, `asyncFib` is _much_ slower than `syncFib`:

![all browsers avg time sync and async result][all-avg-sync-and-async]

Note that this graph only goes from `asyncFib(1)` through `asyncFib(26)`. Across all browsers and devices I've tested, `asyncFib(26)` takes on average 11.5 seconds to complete, while `syncFib` takes on less than 9 _milliseconds_. So much for trying to leverage the multi-threaded nature of browsers.

But upon reflection, this result made sense: Aside from the two recursive calls, `syncFib` is an exceedingly simple function. In contrast, `asyncFib` requires the browser to manage thousands of asynchronous function calls, moving the corresponding messages into and out of the event queue.

### Adding busywork

But I hadn't given up hope to use asynchronicity to make my JavaScript multi-threaded. What if we added busywork prior to the recursive calls to drown out the overhead associated with the event queue? Enter `syncBusyFib` and `asyncBusyFib`:

```javascript
function syncBusyFib(n) {
  if (n <= 0) return 0;
  if (n === 1) return 1;

  const superBig = n ** 9;
  for (let i = 0; i < superBig; i++) {
    i;
  }

  return syncBusyFib(n - 1) + syncBusyFib(n - 2);
}
```

```javascript
async function asyncBusyFib(n) {
  if (n <= 0) return 0;
  if (n === 1) return 1;

  const superBig = n ** 9;
  for (let i = 0; i < superBig; i++) {
    i;
  }

  const prevValues = await Promise.all([
    asyncBusyFib(n - 1),
    asyncBusyFib(n - 2),
  ]);

  return prevValues[0] + prevValues[1];
}
```

It turns out that the calculation times of `syncBusyFib` and `asyncBusyFib` are virtually identical!

![all browsers avg time syncBusy and async result][all-avg-sync-busy-and-async-busy]

(Yes, those are two lines on top of each other!)

There we have it: When we slow down our Fibonacci functions with busywork, the overhead required to execute the async version is drowned out. And clearly, `asyncBusyFib` is executed on a single thread, or else we would expect it to be much faster than `syncBusyFib`.

This impression was borne out when I inspected my CPU activity using [Stacer][stacer]. Here is the CPU activity on a Dell XPS 15 9570 with 12 logical cores (1 CPU with 6 physical cores and hyper-threading) running Ubuntu for `syncBusyFib(13)`:

![Stacer syncBusy Firefox 13][stacer-sync-busy-firefox-13]

(`syncBusyFib` was running during the time when CPU2 hovers around 100%. The flare up of CPU7 happened when I switched to my screenshot tool.)

And here is an almost identical picture for `asyncBusyFib(13)`:

![Stacer asyncBusy Firefox 13][stacer-async-busy-firefox-13]

Our story could end here: We might conclude that we can't leverage the multi-threaded nature of browsers by implementing recursive algorithms with async functions, as the the sync and async versions of an algorithm are at best equally fast, and at worst the async version is much slower.

And I should've expected all of this at the outset: After all, the event queue is a _queue_. Browsers are said to process the messages in the queue sequentially.

### A new hope: Chromium

But something surprising happened when broke down the results by browser. Here are the average calculation times of `syncFib` and `asyncFib` in Firefox and Chromium-based browsers (Chromium, Chrome, and Opera) on the Dell XPS 15 9570 on Ubuntu:

![Linux Firefox Chromium sync and async result][linux-firefox-chromium-sync-and-async]

As we can see, `syncFib` is slightly faster in Firefox than in Chromium-based browsers. But more importantly, `asyncFib` is _much_ faster in Chromium-based browsers than in Firefox: `asyncFib(30)` takes only 9.2 seconds in Chromium-based browsers, compared to a whooping 49.7 seconds in Firefox!

Might the recursive calls actually be called in parallel in Chromium-based browsers? Indeed they are! First, here's the CPU activity for `asyncFib(30)` in Firefox:

![Stacer async Firefox 30][stacer-async-firefox-30]

Firefox seems to use more than one logical core for `asyncFib`, but never more than one at the same time. (The flare up of CPU3 towards the beginning again happened when I switched to my screenshot tool.)

Compare that to the CPU activity for `asyncFib(30)` in Chromium:

![Stacer async Chromium 30][stacer-async-chromium-30]

Chromium uses _all_ 12 logical cores for `asyncFib`! There thus seems to be a fundamental difference between how SpiderMonkey, Firefox's JavaScript engine, and V8, Chromium's JavaScript engine, manage the event queue.

To be sure, `asyncFib` is still much slower in Chromium than `syncFib`. But by now I wasn't surprised by that, given the overhead associated with async functions.

Finally, parallel execution of recursive function was within reach, at least for Chromium-based browsers.

### The revenge of busywork

But it wasn't meant to be. For some reason, Chromium-based browsers _don't_ use multiple cores in parallel for `asyncBusyFib`. Here are the results for `syncBusyFib` and `asyncBusyFib` broken down for Firefox and Chromium-based browsers:

![Linux Firefox Chromium syncBusy and asyncBusy result][linux-firefox-chromium-sync-busy-and-async-busy]

In both Chromium-based browsers and in Firefox, `syncBusy` is as fast as `asyncBusy`. And in Firefox they are even a little bit faster than in Chromium-based browsers.

A look at the CPU activity confirms that Chromium doesn't use multiple cores in parallel to run `asyncBusyFib(13)`:

![Stacer asyncBusy Chromium 13][stacer-async-busy-chromium-13]

(As before, flare up of CPU3 towards the beginning happened when I switched to my screenshot tool.)

For whatever reason, when it matters, V8 decides to forego parallelism.

### Better performance without Web Workers

All results presented so far use [Web Workers][web-workers] to execute the Fibonacci functions. This leads to a much better user experience, since browsers run Web Workers in background threads, which makes it so that the Fibonacci functions don't block the rest of the page. Interestingly, invoking the functions in the main thread without the use of Web Workers is quite a bit faster:

![Linux Firefox Chromium sync and async without Worker result][linux-firefox-chromium-sync-and-async-without-worker]

This difference is particularly significant for `async` in Chromium-based browsers: On the 12-core Dell machine `asyncFib(30)` took on average 4.3 seconds without a Web Worker and 9.3 seconds with a Web Worker.

### What about Web Workers?

Despite this performance hit, it might be wondered whether we should use Web Workers all the way, by having Web Workers spawn subworkers for the recursive call of the Fibonacci function. (Note that Chromium-based browsers require a [polyfill][subworkers] for subworker functionality.)

Unfortunately, there's no way to await a message from a Web Worker. The Web Worker API only provides us with an `onmessage` event that triggers a callback _whenever_ the Web Worker posts a message. It is thus not possible to implement Fibonacci where the recursive calls spawn subworkers. The same is true for the brand new [`worker` module][node-worker-threads] in Node.js.

### Assorted observations

I close with some assorted observations regarding Edge, lower-end devices, Safari, iOS, Android, and memoization.

#### Edge

Here are the average calculation times of `syncFib` and `asyncFib` in Edge, Firefox, and Chromium-based browsers (Chrome and and Opera) on the Dell XPS 15 9570 on Windows:

![Windows Edge Firefox Chromium sync and async result][windows-edge-firefox-chromium-sync-and-async]

As we can see, `syncFib` is equally fast in all browsers, and Chromium-based browsers are again much faster than Firefox when handling `asyncFib`. But more importantly, `asyncFib` is _much_ slower in Edge than in Firefox and Chromium-based browsers: `asyncMemo(25)` took an average of .5 seconds in Chromium-based browsers, 3.9 seconds in Firefox, and 69.5 seconds in Edge!

#### Fewer cores

What does the comparison look like on a lower-end device with fewer logical cores? I tested `syncFib` and `asyncFib` on an early 2015 13-inch MacBook Air with 4 logical cores (1 CPU with 2 physical cores and hyper-threading). Here are the results:

![macOS Firefox Chromium sync and async result][macos-firefox-chromium-sync-and-async]

`syncFib` has almost exactly the same calculation times in Firefox and in Chromium-based browsers. But for `asyncFib`, the picture looks similar for Firefox and Chromium-based browsers as it did on the 12-core Dell machine: `asyncFib(28)` only took 3.5 times as long in Firefox as in Chromium-based browsers (33.1 seconds in Firefox vs. 9.4 seconds in Chromium-based browers).

However, on the 12-core Dell machine running Ubuntu, `asyncFib(28)` took 6 times as long in Firefox as in Chromium-based browsers (18.7 seconds in Firefox vs. 3.1 seconds in Chromium-based browers). And on the 12-core Dell machine running Windows, `asyncFib(28)` took 5.9 times in Firefox (16.4 seconds vs. 2.8 seconds). We'd need to perform the same benchmarks on a 12-core macOS device, or on a 4-core device running Ubuntu or Windows, but these findings suggest that tripling the number of logical cores doesn't triple the speed of `asyncFib` in Chromium-based browsers.

#### Safari

How does Safari fare on the 4-core MacBook Air? Here are the results alongside the previous results for Firefox and Chromium-based browsers:

![macOS Safari Firefox Chromium sync and async result][macos-safari-firefox-chromium-sync-and-async]

(The [`navigator.hardwareConcurrency`][hardware-concurrency] property that I use to read information about the number of logical cores on a device was unavailable in Safari, which is why it says that this is on devices with any number of logical cores. But the Firefox and Chromium-based data is the same here as in the previous chart.)

It turns out that `syncFib` is much faster in Safari: `syncFib(45)` took on average 12.8 seconds in Safari, compared to 25.1 seconds in Chrome and 25.7 seconds in Firefox.

`asyncFib` in Safari is right in between Chromium-based browsers and Firefox: For `asyncFib(28)`, we have 24.7 seconds in Safari vs. the 33.1 seconds in Firefox and 9.4 seconds in Chromium-based browers. I haven't found a tool for macOS that's as powerful at displaying CPU usage as Stacer is on Ubuntu. But I suspect that Safari doesn't use more than one core at once, and Safari's advantage over Firefox with respect to `asyncFib` is due to the same factors as its advantage over Firefox with respect to `syncFib`.

#### iOS

Here's another odd one: I ran `syncFib` and `asyncFib` in Mobile Safari, Firefox Mobile, and Chrome Mobile on a iPad Pro from November 2015. Unfortunately, the `navigator.hardwareConcurrency` property wasn't available in any of these browsers on iOS. But from what I can tell, the A9X CPU inside the iPad Pro has [2 physical cores][a9x], and I expect it to support hyper-threading, thereby giving us 4 logical cores.

Nevertheless, it turns out that Mobile Safari, Firefox Mobile, and Chrome Mobile were completely on a par on the iPad!

![iOS Safari Firefox Chromium sync and async result][ios-safari-firefox-chromium-sync-and-async]

Something seems to prevent Chrome Mobile from using parallelism on iOS.

#### Android

There was a surprise in the other direction when I tested `syncBusy` and `asyncBusy` on a Samsung Galaxy S8 Active with 8 logical cores running Android:

![Android Firefox Chromium sync and async result][android-firefox-chromium-sync-and-async]

`syncFib` was slightly faster in Firefox than in the Chromium-based browsers I tested (Chrome Mobile, Opera Mobile, UC Browser, and Samsung Internet).

But `asyncFib` was _way_ faster in Chromium-based browsers than in Firefox: `asyncFib(27)` took 10.6 times as long in Firefox than in Chromium-based browsers (51.1 seconds vs. 4.8 seconds)! So, the speeup we see in Chromium-based browsers on Android is even more than what we would expect if we assumed that Chromium-based browsers made full use of all 8 logical cores.

#### Memoization

Finally, let's talk about memoization. I tested a sync and an async version of memoized Fibonacci:

```javascript
function syncMemoFib(n, memo = {}) {
  if (n <= 0) return 0;
  if (n === 1) return 1;
  if (memo[n]) return memo[n];

  const first = syncMemoFib(n - 1, memo);
  const second = syncMemoFib(n - 2, memo);
  memo[n] = first + second;

  return memo[n];
}
```

```javascript
async function asyncMemoFib(n, memo = {}) {
  if (n <= 0) return 0;
  if (n === 1) return 1;
  if (memo[n]) return memo[n];

  const prevValues = await Promise.all([
    asyncMemoFib(n - 1, memo),
    asyncMemoFib(n - 2, memo),
  ]);

  memo[n] = prevValues[0] + prevValues[1];
  return memo[n];
}
```

`syncMemoFib` is an `O(n)` function, and an incredibly fast one at that: On the 12-core Dell machine, when invoked with an `n` of 1476 (the largest argument that doesn't return `infinity`), `syncMemoFib` only takes 0.95 milliseconds on average.

I expected `asyncMemoFib` to be much slower than that, but I still expected it to be linear. But oddly enough, the average calculation times of `async` and `asyncMemo` on the 12-core Dell machine running Ubuntu are almost the same in Firefox and _exactly_ the same in Chromium-based browsers:

![Linux Firefox Chromium async and asyncMemo result][linux-firefox-chromium-async-and-async-memo]

I don't even come close to having a suspicion about what's going on here. ¯\\\_(ツ)\_/¯

<!-- Images: -->

[all-avg-sync]: https://www.github.com/m1010j/async-explorations-writeup/raw/master/media/all-avg-sync.png 'all browsers avg time sync result'
[all-avg-sync-and-async]: https://www.github.com/m1010j/async-explorations-writeup/raw/master/media/all-avg-sync-and-async.png 'all browsers avg time sync and async result'
[all-avg-sync-busy-and-async-busy]: https://www.github.com/m1010j/async-explorations-writeup/raw/master/media/all-avg-sync-busy-and-async-busy.png 'all browsers avg time syncBusy and async result'
[stacer-sync-busy-firefox-13]: https://www.github.com/m1010j/async-explorations-writeup/raw/master/media/stacer-sync-busy-firefox-13.png 'Stacer syncBusy Firefox 13'
[stacer-async-busy-firefox-13]: https://www.github.com/m1010j/async-explorations-writeup/raw/master/media/stacer-async-busy-firefox-13.png 'Stacer asyncBusy Firefox 13'
[linux-firefox-chromium-sync-and-async]: https://www.github.com/m1010j/async-explorations-writeup/raw/master/media/linux-firefox-chromium-sync-and-async.png 'Linux Firefox Chromium sync and async result'
[windows-edge-firefox-chromium-sync-and-async]: https://www.github.com/m1010j/async-explorations-writeup/raw/master/media/windows-edge-firefox-chromium-sync-and-async.png 'Windows Edge Firefox Chromium sync and async result'
[stacer-async-firefox-30]: https://www.github.com/m1010j/async-explorations-writeup/raw/master/media/stacer-async-firefox-30.png 'Stacer async Firefox 30'
[stacer-async-chromium-30]: https://www.github.com/m1010j/async-explorations-writeup/raw/master/media/stacer-async-chromium-30.png 'Stacer async Chromium 30'
[linux-firefox-chromium-sync-busy-and-async-busy]: https://www.github.com/m1010j/async-explorations-writeup/raw/master/media/linux-firefox-chromium-sync-busy-and-async-busy.png 'Linux Firefox Chromium syncBusy and asyncBusy result'
[stacer-async-busy-chromium-13]: https://www.github.com/m1010j/async-explorations-writeup/raw/master/media/stacer-async-busy-chromium-13.png 'Stacer asyncBusy Chromium 13'
[macos-firefox-chromium-sync-and-async]: https://www.github.com/m1010j/async-explorations-writeup/raw/master/media/macos-firefox-chromium-sync-and-async.png 'macOS Firefox Chromium sync and async result'
[macos-safari-firefox-chromium-sync-and-async]: https://www.github.com/m1010j/async-explorations-writeup/raw/master/media/macos-safari-firefox-chromium-sync-and-async.png 'macOS Safari Firefox Chromium sync and async result'
[ios-safari-firefox-chromium-sync-and-async]: https://www.github.com/m1010j/async-explorations-writeup/raw/master/media/ios-safari-firefox-chromium-sync-and-async.png 'iOS Safari Firefox Chromium sync and async result'
[android-firefox-chromium-sync-and-async]: https://www.github.com/m1010j/async-explorations-writeup/raw/master/media/android-firefox-chromium-sync-and-async.png 'Android Firefox Chromium sync and async result'
[linux-firefox-chromium-async-and-async-memo]: https://www.github.com/m1010j/async-explorations-writeup/raw/master/media/linux-firefox-chromium-async-and-async-memo.png 'Linux Firefox Chromium async and asyncMemo result'
[linux-firefox-chromium-sync-and-async-without-worker]: https://www.github.com/m1010j/async-explorations-writeup/raw/master/media/linux-firefox-chromium-sync-and-async-without-worker.png 'Linux Firefox Chromium sync and async without Worker result'

<!-- Links: -->

[explorations]: https://async.matthiasjenny.com/
[explorations-results]: https://async.matthiasjenny.com/#results
[api]: https://github.com/m1010j/async-explorations#api
[source]: https://github.com/m1010j/async-explorations
[explorations-benchmarks]: https://async.matthiasjenny.com/#benchmarks
[stacer]: https://github.com/oguzhaninan/Stacer
[web-workers]: https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Using_web_workers
[subworkers]: https://github.com/dmihal/Subworkers
[node-worker-threads]: https://nodejs.org/api/worker_threads.html
[hardware-concurrency]: https://developer.mozilla.org/en-US/docs/Web/API/NavigatorConcurrentHardware/hardwareConcurrency
[a9x]: https://www.anandtech.com/show/9780/taking-notes-with-ipad-pro/2

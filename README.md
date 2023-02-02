# Pending Beacon API

[![Super-Linter](https://github.com/WICG/pending-beacon/actions/workflows/linter.yml/badge.svg)](https://github.com/WICG/pending-beacon/actions/workflows/linter.yml)
[![Spec Prod](https://github.com/WICG/pending-beacon/actions/workflows/auto-publish.yml/badge.svg)](https://github.com/WICG/pending-beacon/actions/workflows/auto-publish.yml)

Authors: [Darren Willis](https://github.com/darrenw), [Fergal Daly](https://github.com/fergald), [Ming-Ying Chung](https://github.com/mingyc) - Google

This document is an explainer for a system for sending beacons when pages are discarded,
that uses a stateful JavaScript API rather than having developers explicitly send beacons themselves.

See also the proposed [spec](https://wicg.github.io/pending-beacon/).

## Problem And Motivation

Web developers have a need for *‘beaconing’* -
that is, sending a bundle of data to a backend server, without expecting a particular response,
ideally at the ‘end’ of a user’s visit to a page.
There are currently
[four major methods](https://calendar.perfplanet.com/2020/beaconing-in-practice/) of beaconing used around the web:

* Adding `<img>` tags inside dismissal events.
* Sending a sync [`XMLHttpRequest`](https://developer.mozilla.org/en-US/docs/Web/API/XMLHttpRequest).
    Note: doesn’t work as part of dismissal events.
* Using the [`Navigator.sendBeacon`][sendBeacon-api] API.
* Using the [`fetch`](https://developer.mozilla.org/en-US/docs/Web/API/fetch) API with the `keepalive` flag.

(There may be other methods; these are the main ones.)

These methods all suffer from reliability problems, stemming from one core issue:
**There is not an ideal time in a page’s lifecycle to make the JavaScript call to send out the beacon.**

* [`unload`](https://developer.mozilla.org/en-US/docs/Web/API/Window/unload_event)
    and [`beforeunload`](https://developer.mozilla.org/en-US/docs/Web/API/Window/beforeunload_event) are unreliable,
    and outright ignored by several major browsers.
* [`pagehide`](https://developer.mozilla.org/en-US/docs/Web/API/Window/pagehide_event)
    and [`visibilitychange`](https://developer.mozilla.org/en-US/docs/Web/API/Document/visibilitychange_event)
    have [issues](https://github.com/w3c/page-visibility/issues/59) on mobile platforms.

To simplify this issue and make beaconing more reliable,
this document proposes adding a stateful JavaScript API where a page can register that it wants a beacon (or beacons) issued when the unloads or hidden.
Developers can populate beacon(s) with data as the user uses the page,
and the browser ensures beacon(s) are reliably sent at some point in time.
This frees developers from worrying about which part of the page lifecycle to send their beacon calls in.

## Goals

Provide a conservatively scoped API,
which allows website authors to specify one or more beacons (HTTP requests)
that should be sent reliably when the page is being unloaded.

## Requirements

* The beacon should be sent at or close to page discard time.
  * For frozen pages that are never unfrozen, this should happen either when the frozen page is removed from memory (BFCache eviction),
    or after a developer-specified timeout
    (using [timeout-related properties](#properties) described below).
  * For browser crashes, forced app closures, etc, the browser should make an effort to send the beacons the next time it is launched
    (guarantees around privacy and reliability here will be the same as the Reporting API’s crash reporting).
* The beacon destination URL should be modifiable.
* The beacon should be visible to (and blockable by) extension,
  to give users control over beacons if they so choose (as they do over current beaconing techniques).

One possible requirement that is missing some clarity is

* The beacon should be cancelable.

This introduces many implementation complications in a multi-process browser.
In order to be resilient to crashes, the beacons must have a presence outside of their process
but in order to be cancellable (without race conditions) the state in process must be authoritative.
If perfectly cancellable beacons are not needed, then the [alternative write-only API](#write-only-api) becomes possible.

## Design

The basic idea is to extend the existing JavaScript [beacon API][sendBeacon-api] by adding a stateful version:

Rather than a developer calling `navigator.sendBeacon`,
the developer registers that they would like to send a beacon for this page when it gets discarded,
and the browser returns a handle to an object that represents a beacon that the browser promises to send on page discard (whenever that is).
The developer can then call methods on this registered beacon handle to populate it with data.

Then, at some point later after the user leaves the page, the browser will send the beacon.
From the point of view of the developer the exact beacon send time is unknown. On successful sending, the whole response will be ignored, including body and headers. Nothing at all should be processed or updated.

### JavaScript API

 In detail, the proposed design includes a new interface [`PendingBeacon`](#pendingbeacon),
 and two of its implementations [`PendingGetBeacon`](#pendinggetbeacon) and [`PendingPostBeacon`](#pendingpostbeacon):

---

#### `PendingBeacon`

`PendingBeacon` defines the common properties & methods representing a beacon.
However, it should not be constructed directly.
Use [`PendingGetBeacon`](#pendinggetbeacon) or [`PendingPostBeacon`](#pendingpostbeacon) instead.

The entire `PendingBeacon` API is only available in [Secure Contexts](https://w3c.github.io/webappsec-secure-contexts/).

##### Properties

The `PendingBeacon` class define the following properties:

* `url`: An immutable `String` property reflecting the target URL endpoint of the pending beacon. The scheme must be **https:** if exists.
* `method`: An immutable property defining the HTTP method used to send the beacon.
  Its value is a `string` matching either `'GET'` or `'POST'`.
* `backgroundTimeout`: A mutable `Number` property specifying a timeout in milliseconds whether the timer starts after the page enters the next `hidden` visibility state.
  If setting the value `>= 0`, after the timeout expires, the beacon will be queued for sending by the browser, regardless of whether or not the page has been discarded yet.
  If the value `< 0`, it is equivalent to no timeout and the beacon will only be sent by the browser on page discarded or on page evicted from BFCache.
  The timeout will be reset if the page enters `visible` state again before the timeout expires.
  Note that the beacon is not guaranteed to be sent at exactly this many milliseconds after `hidden`,
  because the browser has freedom to bundle/batch multiple beacons,
  and the browser might send out earlier than specified value (see [Privacy Considerations](#privacy-considerations)).
  Defaults to `-1`.
* `timeout`: A mutable `Number` property representing a timeout in milliseconds where the timer starts immediately after its value is set or updated.
  If the value `< 0`, the timer won't start.
  Note that the beacon is not guaranteed to be sent at exactly this many milliseconds after `hidden`,
  the browser has freedom to bundle/batch multiple beacons,
  and the browser might send out earlier than specified value (see [Privacy Considerations](#privacy-considerations)).
  Defaults to `-1`.
* `pending`: An immutable `Boolean` property that returns `true` if the beacon has **not** yet started the sending process and has **not** yet been deactivated.
  Returns `false` if it is being sent, fails to send, or deactivated.

Note that attempting to directly assign a value to the immutable properties will have no observable effect.

##### Methods

The `PendingBeacon` class define the following methods:

* `deactivate()`: Deactivate (cancel) the pending beacon.
  If the beacon is already not pending, this won't have any effect.
* `sendNow()`: Send the current beacon data immediately.
  If the beacon is already not pending, this won't have any effect.

---

#### `PendingGetBeacon`

The `PendingGetBeacon` class provides additional methods for manipulating a beacon's GET request data.

##### Constructor

```js
beacon = new PendingGetBeacon(url, options = {});
```

An instance of `PendingGetBeacon` represents a `GET` beacon that will be sent by the browser at some point in the future.
Calling this constructor queues the beacon for sending by the browser;
even if the result goes out of scope,
the beacon will still be sent (unless `deactivate()`-ed beforehand).

The `url` parameter is a string that specifies the value of the `url` property.
It works similar to the existing [`Navigator.sendBeacon`][sendBeacon-api]’s `url` parameter does, except that it only supports https: scheme. The constructor throws a `TypeError` if getting an undefined or a null URL, or a URL of other scheme.

The `options` parameter would be a dictionary that optionally allows specifying the following properties for the beacon:

* `'backgroundTimeout'`
* `'timeout'`

##### Properties

The `PendingGetBeacon` class would support [the same properties](#properties) inheriting from
`PendingBeacon`'s, except with the following differences:

* `method`: Its value is set to `'GET'`.

##### Methods

The `PendingGetBeacon` class would support the following additional methods:

* `setURL(url)`: Set the current beacon's `url` property. The `url` parameter takes a `String`. Throw a `TypeError` if `url` is null, undefined, or has a non https: scheme.

---

#### `PendingPostBeacon`

The `PendingPostBeacon` class provides additional methods for manipulating a beacon's POST request data.

##### Constructor

```js
beacon = new PendingPostBeacon(url, options = {});
```

An instance of `PendingPostBeacon` represents a `POST` beacon.
Simply calling this constructor will **not** queue the beacon for sending.
Instead, a `POST` beacon will **only be queued** by the browser for sending at some point in the future if it has non-`undefined` and non-`null` data.
After it is queued, even if the instance goes out of scope,
the beacon will still be sent (unless `deactivate()`-ed beforehand).

The `url` parameter is a string that specifies the value of the `url` property.
It works similar to the existing [`Navigator.sendBeacon`][sendBeacon-api]’s `url` parameter does, except that it only supports https: scheme. The constructor throws a `TypeError` if getting an undefined or a null URL, or a URL of other scheme.

The `options` parameter would be a dictionary that optionally allows specifying the following properties for the beacon:

* `'backgroundTimeout'`
* `'timeout'`

##### Properties

The `PendingPostBeacon` class would support [the same properties](#properties) inheriting from
`PendingBeacon`'s, except with the following differences:

* `method`: Its value is set to `'POST'`.
* `timeout`: The timer only starts after its value is set or updated **and** `setData(data)` has ever been called with non-`null` and non-`undefined` data.

##### Methods

The `PendingPostBeacon` class would support the following additional methods:

* `setData(data)`: Set the current beacon data.
  The `data` parameter would take the same types as the [sendBeacon][sendBeacon-w3] method’s `data` parameter.
  That is, one of [`ArrayBuffer`][ArrayBuffer-api],
  [`ArrayBufferView`][ArrayBufferView-api], [`Blob`][Blob-api], `String`,
  [`FormData`][FormData-api], or [`URLSearchParams`][URLSearchParams-api].
  If `data` is not `undefined` and not `null`, the browser will queue the beacon for sending,
  which means it kicks off the timer for `timeout` property (if set) and the timer for `backgroundTimeout` property (after the page enters `hidden` state).

---

### Payload

The payload for the beacon will depend on the method used for sending the beacon.
If sent using a POST request, the beacon’s data will be included in the body of the POST request exactly as when [`navigator.sendBeacon`][sendBeacon-api] is used.

For beacons sent via a GET request, there will be no request body.

Requests sent by the pending beacon will include cookies
(the same as requests from [`navigator.sendBeacon`][sendBeacon-api]).

### Extensions

Beacons will be sent with the
[resource type](https://developer.mozilla.org/en-US/docs/Mozilla/Add-ons/WebExtensions/API/webRequest/ResourceType) of ‘beacon’
(or possibly ‘ping’, as Chromium currently sends beacons with the ‘ping’ resource type).
Existing extension APIs that are able to block requests based on their resource types will be able to block these beacons as well.

## Implementation Considerations

This document intentionally leaves out the browser-side implementation details of how beacons will be sent.

This section is here merely to note that there are several considerations browser authors may want to keep in mind:

* Bundling/batching of beacons. Beacons do not need to be sent instantly on page discard,
  and particularly for mobile devices, batching may improve radio efficiency.
* Robustness against crashes/forced terminations/network outages.
* User privacy. See the [Privacy Considerations](#privacy-considerations) section.

### Sync vs Async implementation

The problem with users accessing beacon states, e.g. `pending`, is that it forces us to choose between a synchronous API (that is harder to implement) or an asynchronous API (that is harder to use).

#### Sync implementation (chosen)

With a syncAPI design, the process running JS is authoritative for the state of the beacon
and the following code is correct.

```js
beacon = new PendingBeacon(url, {backgroundTimeout: 1000});
beacon.setData(initialData);
window.setTimeout(() => {
  // By the time this runs, the beacon might have been sent.
  // So check before settings data.
  if (!beacon.pending) {
    beacon = new PendingBeacon(...);
  }
  beacon.setData(newData);
}, someTimeout);
```

However this is harder to implement since the browser now have to coordinate multiple processes
The JS process cannot be the only process involved in the beacon or it will not be crash-resilient and it will also have many of the same problems that an `unload` event handler has.

#### Async implementation

With an async implementation, the [code above](#sync-implementation-chosen) has a race condition.
`pending` may return true but the beacon may be sent immediately after in another process.
This forces us to have an async API where JS can attempt to set new data and is informed afterwards as to whether that succeeded.
E.g.

```js
beacon = new PendingBeacon(url, {backgroundTimeout: 1000});
beacon.setData(initialData);
...
beacon.setData(newData).then(() => {
  // Data was updated successfully.
}).catch(() => {
  // Data was not updated successfully
  beacon = new PendingBeacon(...);
  beacon.setData(newData);
});

```

The code above is *still not correct*.
The call to `setData` does not block and so there may be multiple outstanding calls to `setData`
now their `catch` code has to be coordinated so that only one replacement beacon is created
and the latest data is set on the beacon
(and setting *that* latest data will be async and subject to the same problems).

This is makes it very hard to use the async API correctly.

## Privacy Considerations

This design has limited privacy ramifications above the existing beaconing methods -
it extends the existing beacon API and makes it more reliable.
However, it may break existing means that users have of blocking beaconing -
since the browser itself sends beacons **behind the scenes** (so to speak),
special support may be needed to allow extension authors to block the sending (or registering) of beacons.

Specifically, beacons will have the following privacy requirements:

* Follow third-party cookie rules for beacons.
* Post-unload beacons are not sent if background sync is disabled for a site.
* [#30] Beacons must not leak navigation history to the network provider that it should not know.
  * If network changes after a page is navigated away, i.e. put into bfcache, the beacon should not be sent through the new network;
    If the page is then restored from bfcache, the beacon can be sent.
  * If this is difficult to achieve, consider just force sending out all beacons on navigating away.
* [#27] Beacons must be sent over HTTPS.
* [#34]\[TBD\] Crash Recovery related (if implemented):
  * Delete pending beacons for a site if a user clears site data.
  * Beacons registered in an incognito session do not persist to disk.
* [#3] If a page is suspended (for instance, as part of a [bfcache]),
  beacons should be sent within 30 minutes or less of suspension,
  to keep the beacon send temporally close to the user's page visit.
  Note that beacons lifetime is also capped by the browser's bfcache implementation.

[#3]: https://github.com/WICG/pending-beacon/issues/3
[#27]: https://github.com/WICG/pending-beacon/issues/27
[#30]: https://github.com/WICG/pending-beacon/issues/30
[#34]: https://github.com/WICG/pending-beacon/issues/34
[bfcache]: https://web.dev/bfcache/

## Security Considerations

* What is the maximum size for post beacon data.
* [#27]\[TBD\] Beacons must be sent over HTTPS.
* This is browser-specific implementation detail but the browser process should be careful of the data from `setData(data)` call.

## Alternatives Considered

### DOM-Based API

A DOM-based API was considered as an alternative to this approach.
This API would consist of a new possible `beacon` value for the `rel` attribute
on the link tag, which developers could use to indicate a beacon,
and then use standard DOM manipulation calls to change the data, cancel the beacon, etc.

The stateful JS API was preferred to avoid beacon concerns intruding into the DOM,
and because a ‘DOM-based’ API would still require scripting in many cases anyway
(populating beacon data as the user interacts with the page, for example).

### BFCache-supported `unload`-like event

Another alternative is to introduce (yet) another page lifecycle event,
that would be essentially the `unload` event, but supported by the BFCache -
that is, its presence would not disable the BFCache, and the browser would execute this callback even on eviction from the BFCache.
This was rejected because it would require allowing pages frozen in the BFCache to execute a JavaScript callback,
and it would not be possible to restrict what that callback does
(so, a callback could do things other than sending a beacon, which is not safe).
It also doesn’t allow for other niceties such as resilience against crashes or batching of beacons,
and complicates the already sufficiently complicated page lifecycle.

### Extending Fetch API

Another alternative is to extend the [Fetch API] to support the [requirements](#requirements), of which the following three are critical:

1. A reliable mechanism for delaying operation until page discard, including unloading, an optional timeout after bfcached, bfcache eviction or browser crashes.
2. Doing a keepalive fetch request when that mechanism triggers.
3. Allow pending requests to be updated to reduce network usage.

The existing Fetch with `keepalive` option, combined with `visibilitychagne` listener, can approximate part of (1):

```js
document.addEventListener('visibilitychange', () => {
  if (document.visibilityState === 'hidden') {
    fetch('/send_beacon', {keepalive: true});
    // response may be dropped.
  }
});
```

or a new option `deferSend` may be introduced to cover the entire (1):

```js
// defer request sending on `hidden` or bfcahce eviction etc.
fetch('/send_beacon', {deferSend: true});
// Promise may not resolve and response may be dropped.
```

However, there are several problem with this approach:

1. **The Fetch API shape is not designed for this (1) purpose.** Fundamentally, `window.fetch` returns a Promise with Response to resolve, which don't make sense for beaconing at page discard that doesn't expect to process response.
2. **The (1) mechanism is too unrelated to be added to the Fetch API**. Even just with a new option, bundling it with a visibility event-specific behavior just seems wrong in terms of the API's scope.
3. **The Fetch API does not support updating request URL or data.** This is simply not possible with its API shape. Users have to re-fetch if any update happens.

The above problems suggest that a new API is neccessary for our purpose.

See also discussions in [#52] and [#50].

[#50]: https://github.com/WICG/pending-beacon/issues/50
[#52]: https://github.com/WICG/pending-beacon/issues/52
[Fetch API]: https://fetch.spec.whatwg.org/#fetch-api

### Write-only API

This is similar to the proposed API but there is no `pending` and no `setData()`.
There are 2 classes of beacon with a base class that has

* `url`
* `method`
* `sendNow()`
* `deactivate()`
* API for specifying timeouts

With these APIs, the page cannot check whether the beacon has been sent already.

It's unclear that these APIs can satisfy all use cases.
If they can, they have the advantage of being easier to implement
and simple to use.

### High-Level APIs

#### AppendableBeacon

Has `appendData(data)` which appends new data to the beacon's payload.
The beacon will flush queued payload according to the timeouts and the browser state.

The use-case is for continuously logging events that are accumulated on the server-side.

#### ReplaceableBeacon

Has `replaceData(data)` which replaces the current the beacon's payload.
The beacon will send the payload according to the timeouts and the browser state.
If a payload has been sent already, replaceData simply stores a new payload to be sent in the future.

The use case is for logging a total-so-far.
The server would typically only pay attention to the latest value.

[sendBeacon-api]: https://developer.mozilla.org/en-US/docs/Web/API/Navigator/sendBeacon
[sendBeacon-w3]: https://www.w3.org/TR/beacon/#sec-sendBeacon-method
[ArrayBuffer-api]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/ArrayBuffer
[ArrayBufferView-api]: https://developer.mozilla.org/en-US/docs/Web/API/ArrayBufferView
[Blob-api]: https://developer.mozilla.org/en-US/docs/Web/API/Blob
[FormData-api]: https://developer.mozilla.org/en-US/docs/Web/API/FormData
[URLSearchParams-api]: https://developer.mozilla.org/en-US/docs/Web/API/URLSearchParams

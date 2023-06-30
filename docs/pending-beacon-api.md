# PendingBeacon API (deprecated)

**WARNING: This API is being replaced with [`fetchLater()`](fetch-later-api.md), a Fetch-based approach.**

--

*This document is an explainer for the experimental PendingBeacon API.*
*It describes a system for sending beacons when pages are discarded, rather than having developers explicitly send beacons themselves.*

*Note that the API is avaiable in Chrome as Origin Trial between M107 and M115.*

## Design

The basic idea is to extend the existing JavaScript [beacon API][sendBeacon-api] by adding a stateful version:

Rather than a developer calling `navigator.sendBeacon`,
the developer registers that they would like to send a beacon for this page when it gets discarded,
and the browser returns a handle to an object that represents a beacon that the browser promises to send on page discard (whenever that is).
The developer can then call methods on this registered beacon handle to populate it with data.

Then, at some point later after the user leaves the page, the browser will send the beacon.
From the point of view of the developer the exact beacon send time is unknown. On successful sending, the whole response will be ignored, including body and headers. Nothing at all should be processed or updated.

## JavaScript API

 In detail, the proposed design includes a new interface [`PendingBeacon`](#pendingbeacon),
 and two of its implementations [`PendingGetBeacon`](#pendinggetbeacon) and [`PendingPostBeacon`](#pendingpostbeacon):

---

### `PendingBeacon`

`PendingBeacon` defines the common properties & methods representing a beacon.
However, it should not be constructed directly.
Use [`PendingGetBeacon`](#pendinggetbeacon) or [`PendingPostBeacon`](#pendingpostbeacon) instead.

The entire `PendingBeacon` API is only available in [Secure Contexts](https://w3c.github.io/webappsec-secure-contexts/).

#### Properties

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

#### Methods

The `PendingBeacon` class define the following methods:

* `deactivate()`: Deactivate (cancel) the pending beacon.
  If the beacon is already not pending, this won't have any effect.
* `sendNow()`: Send the current beacon data immediately.
  If the beacon is already not pending, this won't have any effect.

---

### `PendingGetBeacon`

The `PendingGetBeacon` class provides additional methods for manipulating a beacon's GET request data.

#### Constructor

```js
beacon = new PendingGetBeacon(url, options = {});
```

An instance of `PendingGetBeacon` represents a `GET` beacon that will be sent by the browser at some point in the future.
Calling this constructor queues the beacon for sending by the browser;
even if the result goes out of scope,
the beacon will still be sent (unless `deactivate()`-ed beforehand).

The `url` parameter is a string that specifies the value of the `url` property.
It works similar to the existing [`navigator.sendBeacon`][sendBeacon-api]’s `url` parameter does, except that it only supports https: scheme. The constructor throws a `TypeError` if getting an undefined or a null URL, or a URL of other scheme.

The `options` parameter would be a dictionary that optionally allows specifying the following properties for the beacon:

* `'backgroundTimeout'`
* `'timeout'`

#### Properties

The `PendingGetBeacon` class would support [the same properties](#properties) inheriting from
`PendingBeacon`'s, except with the following differences:

* `method`: Its value is set to `'GET'`.

#### Methods

The `PendingGetBeacon` class would support the following additional methods:

* `setURL(url)`: Set the current beacon's `url` property. The `url` parameter takes a `String`. Throw a `TypeError` if `url` is null, undefined, or has a non https: scheme.

---

### `PendingPostBeacon`

The `PendingPostBeacon` class provides additional methods for manipulating a beacon's POST request data.

#### Constructor

```js
beacon = new PendingPostBeacon(url, options = {});
```

An instance of `PendingPostBeacon` represents a `POST` beacon.
Simply calling this constructor will **not** queue the beacon for sending.
Instead, a `POST` beacon will **only be queued** by the browser for sending at some point in the future if it has non-`undefined` and non-`null` data.
After it is queued, even if the instance goes out of scope,
the beacon will still be sent (unless `deactivate()`-ed beforehand).

The `url` parameter is a string that specifies the value of the `url` property.
It works similar to the existing [`navigator.sendBeacon`][sendBeacon-api]’s `url` parameter does, except that it only supports https: scheme. The constructor throws a `TypeError` if getting an undefined or a null URL, or a URL of other scheme.

The `options` parameter would be a dictionary that optionally allows specifying the following properties for the beacon:

* `'backgroundTimeout'`
* `'timeout'`

#### Properties

The `PendingPostBeacon` class would support [the same properties](#properties) inheriting from
`PendingBeacon`'s, except with the following differences:

* `method`: Its value is set to `'POST'`.
* `timeout`: The timer only starts after its value is set or updated **and** `setData(data)` has ever been called with non-`null` and non-`undefined` data.

#### Methods

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

## Extensions

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

In a multi-process browser, in order to be resilient to crashes, the beacons must have a presence outside of their process.
However, in order to allow synchronous mutating operations, e.g. updating or canceling beacons, without introducing data races, the beacon state in process must be authoritative.

The problem with users accessing beacon states, e.g. `pending`, is that it forces implementer to choose between a synchronous API (that is harder to implement) or an asynchronous API (that is harder to use).

If perfectly mutating beacons are not needed, then the [alternative write-only API](alternative-approaches.md#write-only-api) becomes possible.

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

* Beacons should be visible to (and blockable by) extension,
 to give users control over beacons if they so choose (as they do over current beaconing techniques).
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
  * Guarantees around privacy and reliability here should be the same as the Reporting API’s crash reporting
* [#3] If a page is suspended (for instance, as part of a [bfcache]),
  beacons should be sent within 30 minutes or less of suspension,
  to keep the beacon send temporally close to the user's page visit.
  Note that beacons lifetime is also capped by the browser's bfcache implementation.

## Security Considerations

* What is the maximum size for post beacon data.
* [#27]\[TBD\] Beacons must be sent over HTTPS.
* This is browser-specific implementation detail but the browser process should be careful of the data from `setData(data)` call.

## Past Origin Trial in Chrome (M107 - M115)

* [Explanation & Limitation](https://chromium.googlesource.com/chromium/src/+/main/docs/experiments/pending-beacon.md)
* [Dashboard](https://developer.chrome.com/origintrials/#/view_trial/1581889369113886721)
* [Intent to Origin Trial](https://groups.google.com/a/chromium.org/g/blink-dev/c/Vd6RTIfxkiY/m/HECcgiDOAAAJ)
* [Intent to extend OT (M112)](https://groups.google.com/a/chromium.org/g/blink-dev/c/b-XAY59jj0c/m/2jeRBHoMCAAJ)
* [Intent to extend OT (M115)](https://groups.google.com/a/chromium.org/g/blink-dev/c/ZCVcUEYzVHs/m/3PjKPLmkAQAJ)

## Other Documents

* [Chromium Implementation Design Doc](https://groups.google.com/a/chromium.org/g/blink-dev/c/ZCVcUEYzVHs/m/3PjKPLmkAQAJ)


[sendbeacon-api]: https://developer.mozilla.org/en-US/docs/Web/API/Navigator/sendBeacon
[sendBeacon-w3]: https://www.w3.org/TR/beacon/#sec-sendBeacon-method
[ArrayBuffer-api]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/ArrayBuffer
[ArrayBufferView-api]: https://developer.mozilla.org/en-US/docs/Web/API/ArrayBufferView
[Blob-api]: https://developer.mozilla.org/en-US/docs/Web/API/Blob
[FormData-api]: https://developer.mozilla.org/en-US/docs/Web/API/FormData
[URLSearchParams-api]: https://developer.mozilla.org/en-US/docs/Web/API/URLSearchParams
[#3]: https://github.com/WICG/pending-beacon/issues/3
[#27]: https://github.com/WICG/pending-beacon/issues/27
[#30]: https://github.com/WICG/pending-beacon/issues/30
[#34]: https://github.com/WICG/pending-beacon/issues/34
[bfcache]: https://web.dev/bfcache/

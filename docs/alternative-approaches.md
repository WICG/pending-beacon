# Alternative Approaches

*This document maintains the list of alternatives that have been considered to achieve better beaconing.*
*Note that only [PendingBeacon API](#pendingbeacon-api) has ever been implemented.*

## fetch() with PendingRequest API

See the [fetch() with PendingRequest API explainer](fetch-with-pending-request-api.md).

## PendingBeacon API

See the [PendingBeacon API explainer](pending-beacon-api.md).

## DOM-Based API

A DOM-based API was considered as an alternative.
This API would consist of a new possible `beacon` value for the `rel` attribute
on the link tag, which developers could use to indicate a beacon,
and then use standard DOM manipulation calls to change the data, cancel the beacon, etc.

The stateful JS API was preferred to avoid beacon concerns intruding into the DOM,
and because a ‘DOM-based’ API would still require scripting in many cases anyway
(populating beacon data as the user interacts with the page, for example).

## BFCache-supported `unload`-like event

Another alternative is to introduce (yet) another page lifecycle event,
that would be essentially the `unload` event, but supported by the BFCache -
that is, its presence would not disable the BFCache, and the browser would execute this callback even on eviction from the BFCache.
This was rejected because it would require allowing pages frozen in the BFCache to execute a JavaScript callback,
and it would not be possible to restrict what that callback does
(so, a callback could do things other than sending a beacon, which is not safe).
It also doesn’t allow for other niceties such as resilience against crashes or batching of beacons,
and complicates the already sufficiently complicated page lifecycle.

## Extending `fetch()` API

> **NOTE:** Discussions in [#52] and [#50].

Another alternative is to extend the [Fetch API] to support the [requirements](../README.md#requirements).

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

### Problem

However, there are several problem with this approach:

1. **The Fetch API shape is not designed for this (1) purpose.** Fundamentally, `window.fetch` returns a Promise with Response to resolve, which don't make sense for beaconing at page discard that doesn't expect to process response.
2. **The (1) mechanism is too unrelated to be added to the Fetch API**. Even just with a new option, bundling it with a visibility event-specific behavior just seems wrong in terms of the API's scope.
3. **The Fetch API does not support updating request URL or data.** This is simply not possible with its API shape. Users have to re-fetch if any update happens.

The above problems suggest that a new API is neccessary for our purpose.

## Extending `navigator.sendBeacon()` API

> **NOTE:** Discussions in [WebKit's standard position](https://github.com/WebKit/standards-positions/issues/85#issuecomment-1418381239).

Another alternative is to extend the [`navigator.sendBeacon`] API:

```ts
navigator.sendBeacon(url): bool
navigator.sendBeacon(url, data): bool
```

To meet the [requirements](../README.md#requirements) and to make the new API backward compatible, we propose the following shape:

```ts
navigator.sendBeacon(url, data, fetchOptions): PendingBeacon
```

An optional dictionary argument `fetchOptions` can be passed in, which changes the return value from `bool` to `PendingBeacon` proposed in the [above section](#pendingbeacon-api). Some details to note:

1. The proposal would like to support both `POST` and `GET` requests. As the existing API only support `POST` beacons, passing in `fetchOptions` with `method: GET` should enable queuing `GET` beacons.
2. `fetchOptions` can only be a subset of the [Fetch API]'s [`RequestInit`] object:
   1. `method`: one of `GET` or `POST`.
   2. `headers`: supports custom headers, which unblocks [#50].
   3. `body`: **not supported**. POST body should be in `data` argument.
   4. `credentials`: enforcing `same-origin` to be consistent.
   5. `cache`: not supported.
   6. `redirect`: enforcing `follow`.
   7. `referrer`: enforcing same-origin URL.
   8. `referrerPolicy`: enforcing `same-origin`.
   9. `keepalive`: enforcing `true`.
   10. `integrity`: not supported.
   11. `signal`: **not supported**.
       * The reason why `signal` and `AbortController` are not desired is that we needs more than just aborting the requests. It is essential to check a beacon's pending states and to update or accumulate data. Supporting these requirements via the returned `PendingBeacon` object allows more flexibility.
   12. `priority`: enforcing `auto`.
3. `data`: For `GET` beacon, it must be `null` or `undefined`.
4. The return value must supports updating request URL or data, hence `PendingBeacon` object.

### Problem

* The above API itself is enough for the [requirements](../README.md#requirements) (2) and (3), but cannot achieve the requirement (1), delaying the request.
* The function name `sendBeacon` semantic doesn't make sense for the "delaying" behavior.
* Combing the subset of `fetchOptions` along with the existing `data` parameter are error-proning.

## Introducing `navigator.queueBeacon()` API

To imprvoe from "Extending `navigator.sendBeacon()` API, it's better with a new function:

```ts
navigator.queueBeacon(url, fetchOptions, beaconOptions): PendingBeacon
```

This proposal gets rid of the `data` parameter, and request body should be put into `fetchOptions.body` directly.

The extra `beaconOptions` is a dictionary taking `backgroundTimeout` and `timeout` to support the optional timeout after bfcache or hidden requirement.

At the end, this proposal also requires an entirely new API, just under the existing `navigator` namespace. The advantage is that we might be able to merge this proposal into [w3c/beacon] and eliminate the burden to maintain a new spec.


## Write-only API

This is similar to the [proposed PendingBeacon API](#pendingbeacon-api) but there is no `pending` and no `setData()`.
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

## High-Level APIs

### AppendableBeacon

Has `appendData(data)` which appends new data to the beacon's payload.
The beacon will flush queued payload according to the timeouts and the browser state.

The use-case is for continuously logging events that are accumulated on the server-side.

### ReplaceableBeacon

Has `replaceData(data)` which replaces the current the beacon's payload.
The beacon will send the payload according to the timeouts and the browser state.
If a payload has been sent already, replaceData simply stores a new payload to be sent in the future.

The use case is for logging a total-so-far.
The server would typically only pay attention to the latest value.


[#50]: https://github.com/WICG/pending-beacon/issues/50
[#52]: https://github.com/WICG/pending-beacon/issues/52
[Fetch API]: https://fetch.spec.whatwg.org/#fetch-api
[`RequestInit`]: https://fetch.spec.whatwg.org/#requestinit
[w3c/beacon]: https://github.com/w3c/beacon

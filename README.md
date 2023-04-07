# Pending Beacon API

[![Super-Linter](https://github.com/WICG/pending-beacon/actions/workflows/linter.yml/badge.svg)](https://github.com/WICG/pending-beacon/actions/workflows/linter.yml)
[![Spec Prod](https://github.com/WICG/pending-beacon/actions/workflows/auto-publish.yml/badge.svg)](https://github.com/WICG/pending-beacon/actions/workflows/auto-publish.yml)

Authors: [Darren Willis](https://github.com/darrenw), [Fergal Daly](https://github.com/fergald), [Ming-Ying Chung](https://github.com/mingyc) - Google

This document is an explainer for a system for sending beacons when pages are discarded, rather than having developers explicitly send beacons themselves.

## Problem And Motivation

Web developers have a need for *‘beaconing’* -
that is, sending a bundle of data to a backend server, without expecting a particular response,
ideally at the ‘end’ of a user’s visit to a page.
There are currently
[four major methods](https://calendar.perfplanet.com/2020/beaconing-in-practice/) of beaconing used around the web:

* Adding `<img>` tags inside dismissal events.
* Sending a sync [`XMLHttpRequest`].
    Note: doesn’t work as part of dismissal events.
* Using the [`navigator.sendBeacon`] API.
* Using the [`fetch`] API with the `keepalive: true` flag.

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

The following 3 requirements are critical:

1. Support a reliable mechanism for delaying operation until page discard, including unloading.
   1. An optional timeout after visibility: hidden, bfcached, bfcache eviction or browser crashes.
2. Behave like a keepalive fetch request when 1's mechanism triggers.
3. Allow pending requests to be updated to reduce network usage.

### Details

* The beacon should be sent at or close to page discard time.
  * For frozen pages that are never unfrozen, this should happen either when the frozen page is removed from memory (BFCache eviction),
    or after a developer-specified timeout.
  * For browser crashes, forced app closures, etc, the browser should make an effort to send the beacons the next time it is launched
    (guarantees around privacy and reliability here will be the same as the Reporting API’s crash reporting).
* The beacon destination URL should be modifiable.
* The beacon should be visible to (and blockable by) extension,
  to give users control over beacons if they so choose (as they do over current beaconing techniques).

One possible requirement that is missing some clarity is

* The beacon should be updatable after initialization.

This introduces many implementation complications in a multi-process browser.
In order to be resilient to crashes, the beacons must have a presence outside of their process.
However, in order to allow synchronous mutating operations, e.g. update or cancel, without introducing data races, the state in process must be authoritative.
If perfectly mutating beacons are not needed, then the [alternative write-only API](#write-only-api) becomes possible.

## Design

> **NOTE:** Discussions in [#70], [#52] and [#50].

The basic idea is to extend the [Fetch API] by adding a new stateful option:
Rather than a developer manually calling `fetch(url, {keepalive: true})` within a `visibilitychange` event listener, the developer registers that they would like to send a pending request, i.e. a beacon, for this page when it gets discarded.
The developer can then call signal controller registered on this request to updates based on its state or abort.

Then, at some point later after the user leaves the page, the browser will send the request.
From the point of view of the developer the exact send time is unknown. On successful sending, the whole response will be ignored, including body and headers. Nothing at all should be processed or updated, as the page is already gone.

### JavaScript API

The following new fetch options are introduced into [`RequestInit`]:

* `deferSend`: A `DeferSend` object. If set, the browser should defer the request sending until page discard or bfcache eviction.
  Underlying implementation should ensure the request is kept alive until suceeds or fails.
  Hence it cannot work with `keepalive: false`. The object may optionally set the following field:
  * `sendAfterBeingBackgroundedTimeout`: Specifies a timeout in seconds for a timer that only starts after the page enters the next `hidden` visibility state.
    Default to `-1`.
* `sentSignal`: A `SentSignal` object to allow user to listen to the `sent` event of the request when it gets sent.


### Examples

#### Defer a `GET` request until page discard

```js
fetch('/send_beacon', {deferSend: new DeferSend()}).then(res => {
  // Promise may never be resolved and response may be dropped.
})
```

#### Defer a request until next `hidden` + 1 minute

```js
fetch('/send_beacon', {
  deferSend: new DeferSend(sendAfterBeingBackgroundedTimeout: 60)
  }).then(res => {
  // Possibly resolved after next `hidden` + 1 minute.
  // But this may still not be resolved if page is already in bfcache.
})
```

#### Update a pending request

```js
let abort = null;
let pending = true;

function createBeacon(data) {
  pending = true;
  abort = new AbortController();
  let sentSignal = new SentSignal();
  fetch(data, {
    deferSend: new DeferSend(),
    signal: abortController.signal,
    sentSignal: sentsentSignal
  });

  sentSignal.addEventListener("sent", () => {
    pending = false;
  });
}

function updateBeacon(data) {
  if (pending) {
    abort.abort();
  }
  createBeacon(data);
}
```

### Open Discussions

#### 1. Limiting the scope of pending requests

> **NOTE:** Discussions in [#72].

Even if moving toward a fetch-based design, this proposal does still not focus on supporting every type of requests as beacons.

For example, it's non-goal to support most of [HTTP methods], i.e. being able to defer an `OPTION` or `TRACE`.
We should look into [`RequestInit`] and decide whether `deferSend` should throw errors on some of their values:

* `keepalive`: must be `true`. `{deferSend: new DeferSend(), keepalive: false}` conflicts with each other.
* `url`: supported.
* `method`: one of `GET` or `POST`.
* `headers`: supported.
* `body`: only supported for `POST`.
* `signal`: supported.
* `credentials`: enforcing `same-origin` to be consistent.
* `cache`: not supported?
* `redirect`: enforcing `follow`?
* `referrer`: enforcing same-origin URL?
* `referrerPolicy`: enforcing `same-origin`?
* `integrity`: not supported?
* `priority`: enforcing `auto`?

As shown above, at least `keepalive: true` and `method` need to be enforced.
If going with this route, can we also consider the [PendingRequest API] approach that proposes a subclass of `Request` to enforce the above?

#### 2. `sendAfterBeingBackgroundedTimeout` and `deferSend`

> **NOTE:** Discussions in [#73], [#13].

```js
class DeferSend {
  constructor(sendAfterBeingBackgroundedTimeout)
}
```

Current proposal is to make `deferSend` a class, and `sendAfterBeingBackgroundedTimeout` its optional field.

1. Should this be a standalone option in [`RequestInit`]? But it is not relevant to other existing fetch options.
2. Should it be after `hidden` or `pagehide` (bfcached)? (Previous discussion in #13).
3. Need user input for how desirable for this option.
4. Need better naming suggestion.

#### 3. Promise

> **NOTE:** Discussions in [#74].

To maintain the same semantic, browser should resolve Promise when the pending request is sent. But in reality, the Promise may or may not be resolved, or resolved when the page is in bfcache and JS context is frozen. User should not rely on it.

#### 4. `SendSignal`

> **NOTE:** Discussions in [#75].

This is to observe a event to tell if a `deferSend` request is still pending.

To prevent from data races, the underlying implementation should ensure that renderer is authoritative to the request's send state when it's alive. Similar to [this discussion](https://github.com/WICG/pending-beacon/issues/10#issuecomment-1189804245) for PendingBeacon.

#### 5. Handling Request Size Limit

> **NOTE:** Discussions in [#76].

As setting `deferSend` implies `keepalive` is also true, such request has to share the same size limit budget as a regular keepalive request’s [one][fetch-keepalive-quota]: "for each fetch group, the sum of contentLength and inflightKeepaliveBytes <= 64 KB".

To comply with the limit, there are several options:

1. `fetch()` throws `TypeError` whenever the budget has exceeded. Users will not be able to create new pending requests.
2. The browser forces sending out other existing pending requests, in FIFO order, when the budget has exceeded. For a single request > 64KB, `fetch()` should still throws `TypeError`.
3. Ignore the size limit if [BackgroundFetch] Permission is enabled for the page.


#### 6. Permissions Policy

> **NOTE:** Discussions in [#77].

Given that most reporting API providers are crossed origins, we propose to allow this feature by default for 3rd-party iframes.
User should be able to opt out the feature with the corresponding Permissions Policy.


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

### Extending `navigator.sendBeacon()` API

> **NOTE:** Discussions in [WebKit's standard position](https://github.com/WebKit/standards-positions/issues/85#issuecomment-1418381239).

Another alternative is to extend the [`navigator.sendBeacon`] API:

```ts
navigator.sendBeacon(url): bool
navigator.sendBeacon(url, data): bool
```

To meet the [requirements](#requirements) and to make the new API backward compatible, we propose the following shape:

```ts
navigator.sendBeacon(url, data, fetchOptions): PendingBeacon
```

An optional dictionary argument `fetchOptions` can be passed in, which changes the return value from `bool` to `PendingBeacon` proposed in the [`PendingBeacon`-based API](#pendingbeacon-based-api) section. Some details to note:

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

#### Problem

* The above API itself is enough for the requirements (2) and (3), but cannot achieve the requirement (1), delaying the request.
* The function name `sendBeacon` semantic doesn't make sense for the "delaying" behavior.
* Combing the subset of `fetchOptions` along with the existing `data` parameter are error-proning.

### Introducing `navigator.queueBeacon()` API

To imprvoe from "Extending `navigator.sendBeacon()` API, it's better with a new function:

```ts
navigator.queueBeacon(url, fetchOptions, beaconOptions): PendingBeacon
```

This proposal gets rid of the `data` parameter, and request body should be put into `fetchOptions.body` directly.

The extra `beaconOptions` is a dictionary taking `backgroundTimeout` and `timeout` to support the optional timeout after bfcache or hidden requirement.

At the end, this proposal also requires an entirely new API, just under the existing `navigator` namespace. The advantage is that we might be able to merge this proposal into [w3c/beacon] and eliminate the burden to maintain a new spec.

### `PendingBeacon`-based API

> **NOTE**: Offline discussions from [WebKit's standard position](https://github.com/WebKit/standards-positions/issues/85#issuecomment-1418381239), [Fetch-based design][#70] and [PendingRequest API] suggest that a fetch-based approach is preferred.

 This proposal includes a stateful JavaScript API family, a new interface `PendingBeacon` and two of its implementations `PendingGetBeacon` and `PendingPostBeacon`.
 An instance of them represents a pending HTTP request that will be sent by the browser at some point in the future.
 Calling this constructor queues the beacon for sending by the browser;
 even if the result goes out of scope, the beacon will still be sent, unless deactivated beforehand.

 See [previous version of the explainer][pendingbeacon-proposal] for more details.

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

[`XMLHttpRequest`]: https://developer.mozilla.org/en-US/docs/Web/API/XMLHttpRequest
[`fetch`]: https://developer.mozilla.org/en-US/docs/Web/API/fetch
[#3]: https://github.com/WICG/pending-beacon/issues/3
[#13]: https://github.com/WICG/pending-beacon/issues/13
[#27]: https://github.com/WICG/pending-beacon/issues/27
[#30]: https://github.com/WICG/pending-beacon/issues/30
[#34]: https://github.com/WICG/pending-beacon/issues/34
[bfcache]: https://web.dev/bfcache/
[#50]: https://github.com/WICG/pending-beacon/issues/50
[#52]: https://github.com/WICG/pending-beacon/issues/52
[#70]: https://github.com/WICG/pending-beacon/issues/70
[#72]: https://github.com/WICG/pending-beacon/issues/72
[#73]: https://github.com/WICG/pending-beacon/issues/73
[#74]: https://github.com/WICG/pending-beacon/issues/74
[#75]: https://github.com/WICG/pending-beacon/issues/75
[#76]: https://github.com/WICG/pending-beacon/issues/76
[#77]: https://github.com/WICG/pending-beacon/issues/77
[Fetch API]: https://fetch.spec.whatwg.org/#fetch-api
[`RequestInit`]: https://fetch.spec.whatwg.org/#requestinit
[w3c/beacon]: https://github.com/w3c/beacon
[pendingbeacon-proposal]: https://github.com/mingyc/pending-beacon/blob/77291c0d9a98dbe35244df663010ba1f69558451/README.md#javascript-api
[fetch-keepalive-quota]: https://fetch.spec.whatwg.org/#http-network-or-cache-fetch
[BackgroundFetch]: https://developer.mozilla.org/en-US/docs/Web/API/Background_Fetch_API#browser_compatibility
[HTTP methods]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Methods
[PendingRequest API]: https://docs.google.com/document/d/1QQFFa6fZR4LUiyNe9BJQNK7dAU36zlKmwf5gcBh_n2c/edit#heading=h.xs53e9immw2r

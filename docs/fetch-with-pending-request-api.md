# fetch() with PendingRequest API (deprecated)

**WARNING: This API is being replaced with [`fetchLater()`](fetch-later-api.md), a Fetch-based approach.**

--

*This document is an explainer for PendingRequest API.*
*It is proposed in response to a series of [discussions and concerns][concerns] around the experimental [PendingBeacon API](pending-beacon-api.md).*
*There many [open discussions](#open-discussions) within the explainers, which leads to the fetchLater() API.*

*Note that this proposal is NEVER implemented.*

## Design

> **NOTE:** Discussions in [#70], [#52] and [#50].

The basic idea is to extend the [Fetch API] by adding a new stateful option:
Rather than a developer manually calling `fetch(url, {keepalive: true})` within a `visibilitychange` event listener, the developer registers that they would like to send a pending request, i.e. a beacon, for this page when it gets discarded.
The developer can then call signal controller registered on this request to updates based on its state or abort.

Then, at some point later after the user leaves the page, the browser will send the request.
From the point of view of the developer the exact send time is unknown. On successful sending, the whole response will be ignored, including body and headers. Nothing at all should be processed or updated, as the page is already gone.

## JavaScript API

The following new fetch options are introduced into [`RequestInit`]:

* `deferSend`: A `DeferSend` object. If set, the browser should defer the request sending until page discard or bfcache eviction.
  Underlying implementation should ensure the request is kept alive until suceeds or fails.
  Hence it cannot work with `keepalive: false`. The object may optionally set the following field:
  * `sendAfterBeingBackgroundedTimeout`: Specifies a timeout in seconds for a timer that only starts after the page enters the next `hidden` visibility state.
    Default to `-1`.
* `sentSignal`: A `SentSignal` object to allow user to listen to the `sent` event of the request when it gets sent.


## Examples

### Defer a `GET` request until page discard

```js
fetch('/send_beacon', {deferSend: new DeferSend()}).then(res => {
  // Promise may never be resolved and response may be dropped.
})
```

### Defer a request until next `hidden` + 1 minute

```js
fetch('/send_beacon', {
  deferSend: new DeferSend(sendAfterBeingBackgroundedTimeout: 60)
  }).then(res => {
  // Possibly resolved after next `hidden` + 1 minute.
  // But this may still not be resolved if page is already in bfcache.
})
```

### Update a pending request

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

## Open Discussions

### 1. Limiting the scope of pending requests

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

### 2. `sendAfterBeingBackgroundedTimeout` and `deferSend`

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

### 3. Promise

> **NOTE:** Discussions in [#74].

To maintain the same semantic, browser should resolve Promise when the pending request is sent. But in reality, the Promise may or may not be resolved, or resolved when the page is in bfcache and JS context is frozen. User should not rely on it.

### 4. `SendSignal`

> **NOTE:** Discussions in [#75].

This is to observe a event to tell if a `deferSend` request is still pending.

To prevent from data races, the underlying implementation should ensure that renderer is authoritative to the request's send state when it's alive. Similar to [this discussion](https://github.com/WICG/pending-beacon/issues/10#issuecomment-1189804245) for PendingBeacon.

### 5. Handling Request Size Limit

> **NOTE:** Discussions in [#76].

As setting `deferSend` implies `keepalive` is also true, such request has to share the same size limit budget as a regular keepalive request’s [one][fetch-keepalive-quota]: "for each fetch group, the sum of contentLength and inflightKeepaliveBytes <= 64 KB".

To comply with the limit, there are several options:

1. `fetch()` throws `TypeError` whenever the budget has exceeded. Users will not be able to create new pending requests.
2. The browser forces sending out other existing pending requests, in FIFO order, when the budget has exceeded. For a single request > 64KB, `fetch()` should still throws `TypeError`.
3. Ignore the size limit if [BackgroundFetch] Permission is enabled for the page.


### 6. Permissions Policy

> **NOTE:** Discussions in [#77].

Given that most reporting API providers are crossed origins, we propose to allow this feature by default for 3rd-party iframes.
User should be able to opt out the feature with the corresponding Permissions Policy.


## Other Documentation

* [Initial PendingRequest API Proposal](https://docs.google.com/document/d/1QQFFa6fZR4LUiyNe9BJQNK7dAU36zlKmwf5gcBh_n2c/edit#heading=h.powlqxc01y5b)
* [Presntation at WebPerf WG 2023/03/16](https://docs.google.com/presentation/d/1w_v2kn4RxDmGQ76HAHbuWpYMPj7XsHkYOILIkLs9ppY/edit#slide=id.pZ)


[concerns]: https://github.com/WICG/pending-beacon/issues/70
[#13]: https://github.com/WICG/pending-beacon/issues/13
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
[fetch-keepalive-quota]: https://fetch.spec.whatwg.org/#http-network-or-cache-fetch
[BackgroundFetch]: https://developer.mozilla.org/en-US/docs/Web/API/Background_Fetch_API#browser_compatibility
[HTTP methods]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Methods
[PendingRequest API]: https://docs.google.com/document/d/1QQFFa6fZR4LUiyNe9BJQNK7dAU36zlKmwf5gcBh_n2c/edit#heading=h.xs53e9immw2r

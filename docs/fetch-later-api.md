# fetchLater() API

*This document is an explainer for fetchLater() API. It is evolved from a series of [discussions and concerns](https://github.com/WICG/pending-beacon/issues/70) around the experimental PendingBeacon API and the draft PendingRequest API.*

[Draft Specification](https://whatpr.org/fetch/1647/9ca4bda...37a66c9.html)

## Overview

`fetchLater()` is a JavaScript API to request a deferred fetch. Once requested, the deffered request is queued by the browser, and will be invoked in one of the following scenarios:

* The document is destroyed.
* After a user-specified time, even if the document is in bfcache.

The API returns a `FetchLaterResult` that contains a boolean field `activated` that may be updated to tell whether the deferred request has been sent out or not.
On successful sending, the whole response will be ignored, including body and headers. Nothing at all should be processed or updated, as the page is already gone.

Note that from the point of view of the API user, the exact send time is unknown.

### Constraints

* A deferred fetch request body, if exists, has to be a byte sequence. Streaming requests are not allowed.
* The total size of deferred fetch request bodies are limited to 64KB per origin. Exceeding this would immediately be rejected with a QuotaExceeded.

## Key scenarios

### Defer a `GET` request until page is destroyed or evicted from bfcache

No matter the request succeeds or not, the browser will drop the resonse or
error from server, and the caller will not be able to tell.

```js
fetchLater('/send_beacon');
```

### Defer a `POST` request for around 1 minute

> **NOTE**: **The actual sending time is unkown**, as the browser may wait for a longer or shorter period of time, e.g., to optimize batching of deferred fetches.

```js
fetchLater({
  url: '/send_beacon'
  method: 'POST'
  body: getBeaconData(),
}, {activateAfter: 60000 /* 1 minute */});
```

### Send a request when page is abondoned

```js
let beaconResult = null;

function createBeacon(data) {
  if (beaconResult && beaconResult.activated) {
    // Avoid creating duplicated beacon if the previous one is still pending.
    return;
  }

  beaconResult = fetchLater(data, {activateAfter: 0});
}

addEventListener('pagehide', () => createBeacon(...));
addEventListener('visibilitychange', () => {
  if (document.visibilityState === 'hidden') {
    // may be the last chance to beacon, though the user could come back later.
    createBeacon(...);
  }
});
```

### Update a pending request

```js
let beaconResult = null;
let beaconAbort = null;

function updateBeacon(data) {
  const pending = !beaconResult || !beaconResult.activated;
  if (pending && beaconAbort) {
    beaconAbort.abort();
  }

  createBeacon(data);
}

function createBeacon(data) {
  if (beaconResult && beaconResult.activated) {
    // Avoid creating duplicated beacon if the previous one is still pending.
    return;
  }

  beaconAbort = new AbortController();
  beaconResult = fetchLater({
    url: data
    signal: beaconAbort.signal
  });
}
```

### Implement `PendingBeacon` with `fetchLater()`

The following implementation try to simulate the behavior of [`PendingBeacon` API](pending-beacon-api.md#javascript-api) from earlier proposal.

```js
class PendingBeacon {
  #abortController = null;
  #requestInfo = null;
  #activateAfter = null;
  #result = null;

  constructor(requestInfo, activateAfter) {
    this.#requestInfo = requestInfo;
    this.#activateAfter = activateAfter;
    this.#schedule();
  }

  // Schedules a deferred request to send on page destroyed or after page in bfcached + `this.#activateAfter` time.
  #schedule() {
    if (this.#result && this.#result.activated) {
      this.#abortController = null;
    }
    if (this.#abortController) {
      // Cacnel previous pending request.
      this.#abortController.abort();
    }

    this.#abortController = new AbortController();
    this.#requestInfo.signal = this.#abortController.signal;
    #result = fetchLater(this.#requestInfo, {activateAfter: this.#activateAfter});
  }

  // Aborts the deferred request and schedules a new one.
  update(requestInfo) {
    this.#requestInfo = requestInfo;
    this.#schedule();
  }

  // sendNow(): User should directly call `fetch(requestInfo)` instead.
}
```

## Alternatives Considered

### 1. BackgroundSync API

The [Background Synchronization API][backgroundsync-api] allows web applications to defer requests to their service worker to handle at a later time, if the device is offline.

However, to use the API requires the control over a service worker from the top-level window open for the origin, which is impossible for 3rd party iframes that want to perform beaconing.

Note that there are discussions ([#3], [#30]) to address PendingBeacon (or fetchLater)'s privacy requirements by reusing BackgroundSync's access permission.

[backgroundsync-api]: https://github.com/WICG/background-sync/blob/main/explainers/sync-explainer.md#the-api
[#3]: https://github.com/WICG/pending-beacon/issues/3#issuecomment-1531639163
[#30]: https://github.com/WICG/pending-beacon/issues/30#issuecomment-1333869614

### 2. BackgroundFetch API

The [Background Fetch API][backgroundfetch-api] provides a way for service workers to defer processing until a user is connected.

Similar to [BackgroundSync API](#1-backgroundsync-api), using BackGroundFetch also requires the control over a service worker, which is impossible for 3rd party iframes that want to perform beaconing.

[backgroundfetch-api]: https://wicg.github.io/background-fetch/

### 3. Other Alternatives

See also PendingBeacon's [Alternative Approaches](alternative-approaches.md).

## Open Discussions

See [Deferred fetching PR](https://github.com/whatwg/fetch/pull/1647).

## Relevant Discussions

See [the fetch-based-api hotlist](https://github.com/WICG/pending-beacon/issues?q=is%3Aissue+is%3Aopen+label%3Afetch-based-api).

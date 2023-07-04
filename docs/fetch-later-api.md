# fetchLater() API

*This document is an explainer for fetchLater() API. It is evolved from a series of [discussions and concerns](https://github.com/WICG/pending-beacon/issues/70) around the experimental PendingBeacon API and the draft PendingRequest API.*

[Draft Specification](https://whatpr.org/fetch/1647/094ea69...152d725.html)

## Overview

`fetchLater()` is a JavaScript API to request a deferred fetch. Once requested, the deffered request is queued by the browser, and will be invoked in one of the following scenarios:

* The document is destroyed.
* The document is bfcached and not restored after a certain time.

The API returns a `FetchLaterResult` that contains a boolean field `sent` that may be updated to tell whether the deferred request has been sent out or not.
On successful sending, the whole response will be ignored, including body and headers. Nothing at all should be processed or updated, as the page is already gone.

Note that from the point of view of the API user, the exact send time is unknown.

### Constraints

* A deferred fetch request body, if exists, has to be a byte sequence. Streaming requests are not allowed.
* The total size of deferred fetch request bodies are limited to 64KB per origin. Exceeding this would immediately reject with a QuotaExceeded.

## Key scenarios

### Defer a `GET` request until page is destroyed or evicted from bfcache

No matter the request succeeds or not, the browser will drop the resonse or
error from server, and the caller will not be able to tell.

```js
fetchLater('/send_beacon');
```

### Defer a `POST` request until around 1 minute after page is bfcached

> **NOTE**: **The actual sending time is unkown**, as the browser may wait for a longer or shorter period of time, e.g., to optimize batching of deferred fetches.

```js
fetchLater({
  url: '/send_beacon'
  method: 'POST'
  body: getBeaconData(),
}, {backgroundTimeout: 60000 /* 1 minute */});
```

### Send a request when page is abondoned

```js
let beaconResult = null;

function createBeacon(data) {
  if (beaconResult && beaconResult.sent) {
    // Avoid creating duplicated beacon if the previous one is still pending.
    return;
  }

  beaconResult = fetchLater(data, {backgroundTimeout: 0});
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
  const pending = !beaconResult || !beaconResult.sent;
  if (pending && beaconAbort) {
    beaconAbort.abort();
  }

  createBeacon(data);
}

function createBeacon(data) {
  if (beaconResult && beaconResult.sent) {
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
  #backgroundTimeout = null;
  #result = null;

  constructor(requestInfo, backgroundTimeout) {
    this.#requestInfo = requestInfo;
    this.#backgroundTimeout = backgroundTimeout;
    this.#schedule();
  }

  // Schedules a deferred request to send on page destroyed or after page in bfcached + `this.#backgroundTimeout` time.
  #schedule() {
    if (this.#result && this.#result.sent) {
      this.#abortController = null;
    }
    if (this.#abortController) {
      // Cacnel previous pending request.
      this.#abortController.abort();
    }

    this.#abortController = new AbortController();
    this.#requestInfo.signal = this.#abortController.signal;
    #result = fetchLater(this.#requestInfo, {this.#backgroundTimeout});
  }

  // Aborts the deferred request and schedules a new one.
  update(requestInfo) {
    this.#requestInfo = requestInfo;
    this.#schedule();
  }

  // sendNow(): User should directly call `fetch(requestInfo)` instead.
}
```

## Open Discussions

See [Deferred fetching PR](https://github.com/whatwg/fetch/pull/1647).

## Relevant Discussions

See [the fetch-based-api hotlist](https://github.com/WICG/pending-beacon/issues?q=is%3Aissue+is%3Aopen+label%3Afetch-based-api).

# fetchLater() API

*This document is an explainer for fetchLater() API. It is evolved from a series of [discussions and concerns](https://github.com/WICG/pending-beacon/issues/70) around the experimental PendingBeacon API and the draft PendingRequest API.*

* [Specification PR](https://github.com/whatwg/fetch/pull/1647)
* [Draft Specification](https://whatpr.org/fetch/1647/9ca4bda...37a66c9.html)

## Motivation

See [Motivation - Pending Beacon API](../README.md#motivation).

## Overview

`fetchLater()` is a JavaScript API to request a deferred fetch. Once requested, the deffered request is queued by the browser, and will be invoked in one of the following scenarios:

* The document is destroyed.
* After a user-specified time, even if the document is in bfcache.
* Browser decides its time to send it.

The API returns a `FetchLaterResult` that contains a read-only boolean field `activated` that may be updated by Browser to tell whether the deferred request has been sent out or not.
On successful sending, the whole response will be ignored, including body and headers. Nothing at all should be processed or updated, as the page is already gone.

Note that from the point of view of the API user, the exact send time is unknown.

### Constraints

* A deferred fetch request body, if exists, has to be a byte sequence. Streaming requests are not allowed.
* A new permissions policy `deferred-fetch` is defined to control the feature availability and to delegate request quota. See [Permissions Policy and Quota](#permissions-policy-and-quota).

## Key scenarios

### Defer a `GET` request until page is destroyed or evicted from bfcache

No matter the request succeeds or not, the browser will drop the response or
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

## Permissions Policy and Quota

This section comes from the discussion in [#87].

[#87]: https://github.com/WICG/pending-beacon/issues/87#issuecomment-1985358609

### Permissions Policy: `deferred-fetch`

* Define a new Permissions Policy `deferred-fetch`, default to `self`.
* Every top-level document has a quota of **640KB** for all fetchLater request bodies from its descendants and itself.
* Every reporting origin within a top-level document has a quota of **64KB** across all fetchLater request bodies the document can issue.
* A cross-origin child document is only allowed to make fetchLater requests if its origin is allowed by its top-level document’s `deferred-fetch` policy.

Both quotas may subject to change if we have more developer feedback.

### Default Behavior

Without any configuration, a top-level document can make an unlimited number (N) of fetchLater requests,
but the total of their body size (X1+X2+ … +XN) of the pending fetchLater requests must <= 64KB for a single reporting origin, and <= 640KB across all reporting origins.

```html
<!-- In a top-level document from https://a.com -->
<script>
  fetchLater("https://a.com", {method: "POST", body: "<X1-bytes data>"});
  fetchLater("https://a.com", {method: "POST", body: "<X2-bytes data>"});
  fetchLater("https://b.com", {method: "POST", body: "<X3-bytes data>"});
  fetchLater("https://c.com", {method: "POST", body: "<X4-bytes data>"});

  fetchLater("https://a.com", {method: "GET"});
</script>
```

In the above example, the following requirements must be met:

* Quota for all request bodies X1+X2+X3+X4 <= 640KB
* Quota for request bodies for the origin `https://a.com` X1+X2 <= 64KB
* Quota for request bodies for the origin `https://b.com` X3 <= 64KB
* Quota for request bodies for the origin `https://c.com` X4 <= 64KB

Note that only the size of a POST body counts for the total limit.

### Delegating Quota to Sub-frames

A top-level document can grant additional origins in its descendant to make fetchLater calls by the permissions policy [`deferred-fetch`][deferred-fetch],
which also grants **and shares** the same quota to every of them.
For example, the following iframes “frame-b” and “frame-c” all share the same quota from the their root document:

[deferred-fetch]: https://github.com/w3c/webappsec-permissions-policy/issues/544

```html
<!--
In a top-level document from https://a.com

Permissions-Policy: deferred-fetch=(self "https://b.com" "https://c.com")
-->

<script>
  fetchLater("https://a.com", {method: "POST", body: "<X1-bytes data>"});
  fetchLater("https://b.com", {method: "POST", body: "<X2-bytes data>"});
  fetchLater("https://c.com", {method: "POST", body: "<X3-bytes data>"});
</script>

<iframe id="frame-b" src="https://b.com/iframe" allow="deferred-fetch 'self'">
  <!-- In https://b.com/iframe -->
  <script>
    fetchLater("https://a.com", {method: "POST", body: "<X4-bytes data>"});
    fetchLater("https://b.com", {method: "POST", body: "<X5-bytes data>"});
    fetchLater("https://c.com", {method: "POST", body: "<X6-bytes data>"});
  </script>
</iframe>
<iframe id="frame-c" src="https://c.com/iframe" allow="deferred-fetch 'self'">
  <!-- In https://c.com/iframe -->
  <script>
    fetchLater("https://a.com", {method: "POST", body: "<X7-bytes data>"});
    fetchLater("https://b.com", {method: "POST", body: "<X8-bytes data>"});
    fetchLater("https://c.com", {method: "POST", body: "<X9-bytes data>"});
  </script>
</iframe>
```

In the above example, the following requirements must be met:

* Quota for all request bodies X1+X2+...+X9 <= 640KB
* Quota for request bodies for origin `https://a.com` X1+X4+X7 <= 64KB
* Quota for request bodies for origin `https://b.com` X2+X5+X8 <= 64KB
* Quota for request bodies for origin `https://c.com` X3+X6+X9 <= 64KB

## Security and Privacy

For a high-level overview, see [Self-Review Questionnaire: Security and Privacy](security-privacy-questionnaire.md).

This design has no impact on the existing fetch API.
However, the following security & privacy requirements have been discussed on GitHub and are important to follow:

### Security Considerations

* Deferred requests must be sent over HTTPS. See [security review feedback #27][#27].

[#27]: https://github.com/WICG/pending-beacon/issues/27

### Privacy Considerations

* Deferred requests can only be sent after the page becomes inactive, i.e. bfcached, if BackgroundSync permission is enabled for the Origin of the page. See [privacy review feedback #30][#30].

### Implementation-Specific Considerations

Implementation-specific considerations are not listed in this explainer.
Please refer to each browser implementation design for more details:

* [Chromium](https://docs.google.com/document/d/1U8XSnICPY3j-fjzG35UVm6zjwL6LvX6ETU3T8WrzLyQ/edit#heading=h.kztg1uvdyoki)


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

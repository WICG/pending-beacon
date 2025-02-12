# fetchLater() API

*This document is an explainer for fetchLater() API. It is evolved from a series of [discussions and concerns](https://github.com/WICG/pending-beacon/issues/70) around the experimental PendingBeacon API and the draft PendingRequest API.*

* [Specification PR](https://github.com/whatwg/fetch/pull/1647)
* [Draft Specification](https://whatpr.org/fetch/1647/9ca4bda...37a66c9.html)

## Motivation

See [Motivation - Pending Beacon API](../README.md#motivation).

## Overview

`fetchLater()` is a JavaScript API to request a deferred fetch. Once requested, the deferred request is queued by the browser, and will be invoked in one of the following scenarios:

* The document is destroyed.
* After a user-specified time, even if the document is in bfcache.
* Browser decides its time to send it.

The API returns a `FetchLaterResult` that contains a read-only boolean field `activated` that may be updated by Browser to tell whether the deferred request has been sent out or not.
On successful sending, the whole response will be ignored, including body and headers. Nothing at all should be processed or updated, as the page is already gone.

Note that from the point of view of the API user, the exact send time is unknown.

### Constraints

* A deferred fetch request body, if exists, has to be a byte sequence. Streaming requests are not allowed.
* A new permissions policy `deferred-fetch` is defined to control the feature availability and to delegate request quota. See [Quota and permissions policy](#quota-and-permissions-policy).

## Key scenarios

### Defer a `GET` request until page is destroyed or evicted from bfcache

No matter the request succeeds or not, the browser will drop the response or
error from server, and the caller will not be able to tell.

```js
fetchLater('/send_beacon');
```

### Defer a `POST` request for around 1 minute

> **NOTE**: **The actual sending time is unknown**, as the browser may wait for a longer or shorter period of time, e.g., to optimize batching of deferred fetches.

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
      // Cancel previous pending request.
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

## Quota and Permissions Policy

### Outline

Deferred fetches are different from normal fetches, due to the fact that they are batched and sent once the tab is closed, and at that point the user has no way to abort them.
To avoid situations where documents abuse this bandwidth to send unlimited amounts of data over the network, the overall quota for a top level document is capped at 640KB (which should be enough for anyone).
Since this cap makes deferred fetch bandwidth a scarce resource which needs to be shared between multiple reporting origins (e.g. several RUM libraries) and also across subframes of multiple origins, the platform
provides a reasonable default division of this quota, and also provides knobs, in the form of permission policies, to allow dividing it in a different way when desired.

### Default Behavior

Without any configuration, a top-level document and its same-origin descendant subframes can invoke an unlimited number of `fetchLater` requests, but with the following limitations:

1. The total bandwidth taken by these requests (counting the URL, custom headers and POST body size) must not exceed 64KB for each reporting origin.
1. The total bandwidth for all the reporting origins must not exceed 512KB.

```html
<!-- In a top-level document from https://a.com -->
<script>
  fetchLater("https://a.com", {method: "POST", body: "<16KB data>"});
  fetchLater("https://a.com", {method: "POST", body: "<16KB data>"});
  fetchLater("https://b.com", {method: "POST", body: "<48KB data>"});
  fetchLater("https://c.com", {method: "POST", body: "<1KB data>"});

  fetchLater("https://a.com", {method: "GET"});
</script>
```

In the above example, the following requirements must be met:

* Quota for all request bodies `(13+16+13+16+13+48+13+1+13)KB <= 512KB`.
* Quota for request bodies for the origin `https://a.com` `(13+16+13+16+13)KB <= 64KB`.
* Quota for request bodies for the origin `https://b.com` `13KB+48KB <= 64KB`.
* Quota for request bodies for the origin `https://c.com` `13KB+1KB <= 64KB`.

Note that the size of the URL and additional headers are added to the POST body when counting the total limit, to avoid a situation where data is encoded into the URL to circumvent the limitation.

### Delegating quota to subframes

By default, each cross-origin subframe, together with its same-origin descendants, is granted a deferred-fetching quota of 8KB. This is limited to the first 16 cross-origin iframes, with a total of 128KB.
The top-level page can use permissions policy to tweak this quota: either increase an iframe's quota to 64KB, or revoke it in favor of other iframes. The top-level origin can also revoke this entire 128KB quota
in favor of its own deferred fetches.

An iframe is granted its quota upon being navigated from its parent, based on its permission policy and remaining quota at that time. The quota is reserved for this iframe until its navigable is destroyed (e.g. the iframe is removed from the DOM), and the iframe's owner cannot observe whether the iframe's document or its descendants are using the quota in practice.

By default, a subframe does not share its quota with descendant ("grandchildren" of the top level) cross-origin subframes.
The subframes can use the same permission policies to grant part of the quota or all of it further down to descendant cross-origin subframes.

### Permissions Policy: `deferred-fetch-full` and `deferred-fetch-minimal`

The `deferred-fetch` and `deferred-fetch-minimal` policies determine how the overall 640KB is distributed between the top level origin and its cross-origin subframes.
As mentioned before, by default the top level origin is granted 512KB and each cross-origin subframe is granted 8KB out of the rest of the 128KB.

* The `deferred-fetch`, defaults to `self`, defines whether frames of this origin are granted the full quota for deferred fetching.
* The `deferred-fetch-minimal`. defaults to `*`, defines whether the frame is granted 8KB out of its parent's quota by default.
* A top level frame that has the `deferred-fetch-minimal` permission set to `self` or `()`, does not delegates the minimal 8kb quota to subframes at all. Instead, the 128KB quota for iframes is added to its normal quota.
* A cross-origin subframe that is granted a `deferred-fetch` permission, receives 64KB out of its parent's main quota, if the full 64KB are available at the time of it's container-initiated navigation.
* A cross-origin subframe can grant `deferred-fetch` to one of its cross-origin subframe descendants, delegating its entire quota. This only works if the quota is not used at all.
* A cross-origin subframe cannot grant `deferred-fetch-minimal` to its descendants.
* Permission policy checks are not discernable from quota checks. Calling `fetchLater` will throw a `QuotaExceededError` regardless of the reason.

Note: because of the nature of the permission policy API, documents would have to be granted the most relaxed policy needed for their descendants, and then restrict it per subframe.
Permissions policy don't have semantics to have a strict default and relax it per subframe.

See [`deferred-fetch` permissions policy issue](https://github.com/w3c/webappsec-permissions-policy/issues/544)

#### Usage example

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

#### Quota delegation examples

##### Using up the `minimal` quota

```
Permissions-Policy: deferred-fetch=(self "https://b.com")
```

1. A subframe of `b.com` receives 64KB upon creation.
1. A subframe of `c.com` receives 8KB upon creation.
1. 15 more subframes of different origins would receive `8KB` upon creation.
1. The next subframe would not be greanted any quota.
1. One of the subrames is removed. Its deferred fetches are sent.
1. The next subframe would receive an 8KB quota again.

##### Revoking the `minimal` quota altogether

```
Permissions-Policy: deferred-fetch=(self "https://b.com")
Permissions-Policy: deferred-fetch-minimal=()
```

1. A subframe of `b.com` receives 64KB upon creation.
1. A subframe of `c.com` receives no quota upon creation.
1. The top-level document and its same-origin descendants can use up the full 640KB.

##### Delegating quota from a subframe to its own subframes

```
# Top level
Permissions-Policy: deferred-fetch=(self "https://b.com" "http://c.com" "https://d.com")

# b.com
Permissions-policy: deferred-fetch-minimal=*

# c.com
Permissions-policy: deferred-fetch=(self "https://d.com")
```

1. A subframe with `b.com` would be allowed to use 64KB.
1. A subframe with `c.com` would be allowed to use 64KB.
1. If `c.com` has a `d.com` subframe, and `c.com` hasn't used any of its quota, its quota would be reserved for the `d.com` subframe.

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

Similar to [BackgroundSync API](#1-backgroundsync-api), using BackGroundFetch also requires the control over a service worker, which is impossible for third-party iframes that want to perform beaconing.

[backgroundfetch-api]: https://wicg.github.io/background-fetch/

### 3. Other Alternatives

See also PendingBeacon's [Alternative Approaches](alternative-approaches.md).

## Open Discussions

See [Deferred fetching PR](https://github.com/whatwg/fetch/pull/1647).

## Relevant Discussions

See [the fetch-based-api hotlist](https://github.com/WICG/pending-beacon/issues?q=is%3Aissue+is%3Aopen+label%3Afetch-based-api).

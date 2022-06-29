# Stateful JavaScript Page Unload Beacon API


Authors: [Darren Willis](https://github.com/darrenw), [Fergal Daly](https://github.com/fergald), [Ming-Ying Chung](https://github.com/mingyc) - Google


This document is an explainer for a system for sending beacons when pages are discarded, that uses a stateful API rather than having developers explicitly send beacons themselves.


## Problem And Motivation

Web developers have a need for *‘beaconing’* - that is, sending a bundle of data to a backend server, without expecting a particular response, ideally at the ‘end’ of a user’s visit to a page. There are currently [four major methods](https://calendar.perfplanet.com/2020/beaconing-in-practice/) of beaconing used around the web:

*   Adding `<img>` tags inside dismissal events.
*   Sending a sync [`XMLHttpRequest`](https://developer.mozilla.org/en-US/docs/Web/API/XMLHttpRequest). Note: doesn’t work as part of dismissal events.
*   Using the [`Navigator.sendBeacon`][sendBeacon-api] API.
*   Using the [`fetch`](https://developer.mozilla.org/en-US/docs/Web/API/fetch) API with the `keepalive` flag.

(There may be other methods; these are the main ones.)

These methods all suffer from reliability problems, stemming from one core issue: **There is not an ideal time in a page’s lifecycle to make the JavaScript call to send out the beacon.**

*   [`unload`](https://developer.mozilla.org/en-US/docs/Web/API/Window/unload_event) and [`beforeunload`](https://developer.mozilla.org/en-US/docs/Web/API/Window/beforeunload_event) are unreliable, and outright ignored by several major browsers.
*    [`pagehide`](https://developer.mozilla.org/en-US/docs/Web/API/Window/pagehide_event) and [`visibilitychange`](https://developer.mozilla.org/en-US/docs/Web/API/Document/visibilitychange_event) have [issues](https://github.com/w3c/page-visibility/issues/59) on mobile platforms.

To simplify this issue and make beaconing more reliable, this document proposes adding a stateful JavaScript API where a page can register that it wants a beacon (or beacons) issued when the unloads or hidden. Developers can populate beacon(s) with data as the user uses the page, and the browser ensures beacon(s) are reliably sent at some point in time. This frees developers from worrying about which part of the page lifecycle to send their beacon calls in.


## Goals

Provide a conservatively scoped API, which allows website authors to specify one or more beacons (HTTP requests) that should be sent reliably when the page is being unloaded.


## Requirements

*   The beacon should be sent at or close to page discard time.
    *   For frozen pages that are never unfrozen, this should happen either when the frozen page is removed from memory (BFCache eviction), or after a developer-specified timeout (using the `'pageHideTimeout'` described below)
    *   For browser crashes, forced app closures, etc, the browser should make an effort to send the beacons the next time it is launched (guarantees around privacy and reliability here will be the same as the Reporting API’s crash reporting).
*   The beacon destination URL should be modifiable.
*   The beacon should be cancelable.
*   The beacon should be visible to (and blockable by) extension, to give users control over beacons if they so choose (as they do over current beaconing techniques).


## Design

The basic idea is to extend the existing JavaScript beacon API by adding a stateful version. Rather than a developer calling `navigator.sendBeacon`, the developer registers that they would like to send a beacon for this page when it gets discarded, and the browser returns a handle to an object that represents a beacon that the browser promises to send on page discard (whenever that is). The developer can then call methods on this registered beacon handle to populate it with data. Then, at some point later after the user leaves the page, the browser will send the beacon. From the point of view of the developer the exact beacon send time is unknown.

### JavaScript API


#### Constructor

 In detail, the proposed design is a new class `PendingBeacon`, constructed like so:


```
beacon = new PendingBeacon(url, options = {});
```

An instance of `PendingBeacon` represents a beacon that will be sent by the browser at some point in the future. Calling this constructor queues the beacon for sending by the browser; even if the result goes out of scope, the beacon will still be sent (unless `deactivate()`-ed beforehand).

The `url` parameter is the same as the existing [`Navigator.sendBeacon`][sendBeacon-api]’s `url` parameter. Note that multiple instances of `PendingBeacon` can be made, so multiple beacons can be sent to multiple url endpoints.

The `options` parameter would be a dictionary that optionally allows specifying the `'method'` and `'pageHideTimeout'` properties for the beacon (described below).


#### Methods & Properties

The `PendingBeacon` class would support the following methods/properties:

| *Method/Property Name* | *Description* |
| ---------------------- | ------------- |
| `url` | A [`USVString`][USVString] property reflecting the target URL endpoint of the pending beacon. Can be reset to point the beacon to a new endpoint.      |
| `method` | A property defining the HTTP method used to send the beacon. Its value is a string matching either `'GET'` or `'POST'`. Default to `'POST'`. |
| `deactivate()` | Deactivate (cancel) the pending beacon. |
| `setData(data)` | Set the current beacon data. The `data` argument would take the same types as the [sendBeacon][sendBeacon-w3] method’s `data` parameter. That is, one of [`ArrayBuffer`][ArrayBuffer-api], [`ArrayBufferView`][ArrayBufferView-api], [`Blob`][Blob-api], [`USVString`][USVString], [`FormData`][FormData-api], or [`URLSearchParams`][URLSearchParams-api]. (See the [API Discussion: Failure Handling](#discussion-failure-handling) section for possible behaviors.) |
| `sendNow()` | Send the current beacon data immediately. (See the [API Discussion: Failure Handling](#discussion-failure-handling) section for possible behaviors.) |
| `pageHideTimeout` | Defaults to `null`. If set, a timeout in milliseconds after the next `pagehide` event is sent, after which a beacon will be queued for sending, regardless of whether or not the page has been discarded yet. If this is null when the page is hidden, the beacon will be sent on page discard (including eviction from the BFCache). Note that the beacon is not guaranteed to be sent at exactly this many milliseconds after pagehide; bundling/batching of beacons is possible. |
| `isPending` | A property that returns whether the beacon is still ‘pending’; that is, whether or not the beacon has started the sending process. (See the [API Discussion: Failure Handling](#discussion-failure-handling) section for possible behaviors.) |


### API Discussion: Failure Handling

One fundamental question to this API design is how the failure should be handled. Consider the following use cases:

*   Users would like to ensure the beacon is sent successfully. But `sendNow()` or the browser may fail to send beacon requests.
*   Users would like to update the beacon's data or property before sending. But `setData()`, `url`, `method`, or `pageHideTimeout` may fail to update, if called after the beacon is being sent.


#### By User (API Provides Minimal Control)

In this proposal, the browser should be responsible for retrying failed requests, and the update-related APIs should be always available. Users have no access to beacon states.

*   When a send request fails, no matther it is a `sendNow()` or a browser request, the browser should make attempts to retry the request.
    *  A `retries` can be provided in the constructor to set the maximum number of attempts.
    *  An exception can still be thrown on `sendNow()` failure, in case users want to be aware. Note that exceptions cannot be thrown on page discard.
*   Updating to an existing beacon would never fail by making every instance of `PendingBeacon` reusable:
    *   If a beacon is sent, it can still be updated and sent again.
    *   Whether a beacon should be re-sent on page discard or `pageHideTimeout` after calling `sendNow()` can be configurable.
*   Users have no access to beacon state like `isPending`, request failed, or request sent.

Specifically for `setData()`, there are 2 usage patterns the API can support:


##### `mode: 'replace'`

The data is always fully replaced. It is OK to send the data at any time and the browser should ensure that the final value is always delivered. This could be used to communicate the total-so-far of some metrics.

```
beacon.setData('123')
beacon.sendNow()  // Sends out '123'
beacon.sendNow()  // Sends out '123'
```


##### `mode: 'append'`

The data is always appended-to. A send request here means flush any unsent data. It's OK to send at any time and we want to ensure that everything up to the last appended value is delivered.

```
beacon.setData('123')
beacon.sendNow()  // Sends out '123'
beacon.sendNow()  // Sends out nothing
```


The overall usage would be very simple. Users could trigger any calls whenever they want.

The advantage of this proposal is that users can just keep using the same beacon forever and never worry about whether it is already sent. Sending is 100% write-only with no feedback to JavaScript at all. It works best for the send-and-forget use case.


#### By User (API Provides Fine-Grained Control)

In this proposal, the browser is only responsible for making requests, and the update-related APIs may not work as expected per beacon state. Users have access to some beacon states and have to handle them.

*   When a send request fails,
    *   If it is a `sendNow()`, the API may inform users of the beacon state by
        *  A bool return value (for sync API).
        *  A promise (for async API).
        *  An exception.
    *   If it is a browser request, users may only approximate by some beacon state `isFailed`.
*   Updating to an existing beacon may fail. Users have to rely on the beacon state `isPending` to know if the beacon is sent.


##### Race Condition in Sync & Async API

The problem with users accessing beacon states is that a state can be outdated right after access, due to the fact that a beacon can be sent by the browser at any time.


###### Sync API

In sync API design, race can happen between any calls:

```
beacon = new PendingBeacon(url, {pageHideTimeout: 1000});
if (beacon.isPending) {  // true, as `beacon` is just created.
  // At this line, the page goes into BFCache as the user navigates away.
  // The browser sends out `beacon` after 1 second timeout.
  // The user navigates back, and the page is unfrozen.
  beacon.setData(newData);
}
```


###### Async API

In async API design, race can still happen after a promise resolves.

```
beacon = new PendingBeacon(url, {pageHideTimeout: 1000});
beacon.isPending().then(() => {
  // At this line, the page goes into BFCache as the user navigates away.
  // The browser sends out `beacon` after 1 second timeout.
  // The user navigates back, and the page is unfrozen.
  beacon.setData(newData);
});
```

This may be solved by restricting when the browser is allowed to send or by dropping `isPending` and just relying on async `setData()` to resolve/reject.


### Payload

The payload for the beacon will depend on the method used for sending the beacon. If sent using a POST request, the beacon’s data will be included in the body of the POST request exactly as when [`navigator.sendBeacon`][sendBeacon-api] is used.

For beacons sent via a GET request, the data will be encoded as query parameters in form application/x-www-form-urlencoded.

Requests sent by the pending beacon will include cookies (the same as requests from [`navigator.sendBeacon`][sendBeacon-api]).

### Extensions

Beacons will be sent with the [resource type](https://developer.mozilla.org/en-US/docs/Mozilla/Add-ons/WebExtensions/API/webRequest/ResourceType) of ‘beacon’ (or possibly ‘ping’, as Chromium currently sends beacons with the ‘ping’ resource type). Existing extension APIs that are able to block requests based on their resource types will be able to block these beacons as well.


## Implementation Considerations

This document intentionally leaves out the browser-side implementation details of how beacons will be sent, this section is here merely to note that there are several considerations browser authors may want to keep in mind:

*   Bundling/batching of beacons. Beacons do not need to be sent instantly on page discard, and particularly for mobile devices, batching may improve radio efficiency.
*   Robustness against crashes/forced terminations/network outages.
*   User privacy. See the [Privacy](#privacy) section.


## Privacy

This design has limited privacy ramifications above the existing beaconing methods - it extends the existing beacon API and makes it more reliable. However, it may break existing means that users have of blocking beaconing - since the browser itself sends beacons ‘behind the scenes’ (so to speak), special support may be needed to allow extension authors to block the sending (or registering) of beacons.

Specifically, beacons will have the following privacy requirements:
*   Beacons must be sent over HTTPS.
*   Beacons are only sent over the same network that was active when the beacon was registered (e.g. if the user goes offline and moves to a new network, discard pending beacons).
*   Delete pending beacons for a site if a user clears site data.
*   Beacons registered in an incognito session do not persist to disk.
*   Follow third-party cookie rules for beacons.
*   Post-unload beacons are not sent if background sync is disabled for a site.
*   If a page is suspended (for instance, as part of a [bfcache](https://web.dev/bfcache/)), beacons should be sent within 10 minutes or less of suspension, to keep the beacon send temporally close to the user's page visit.


## Alternatives Considered

**DOM-Based API**

A DOM-based API was considered as an alternative to this approach. This API would consist of a new possible ‘beacon’ value for the ‘rel’ attribute on the link tag, which developers could use to indicate a beacon, and then use standard DOM manipulation calls to change the data, cancel the beacon, etc.

The stateful JS API was preferred to avoid beacon concerns intruding into the DOM, and because a ‘DOM-based’ API would still require scripting in many cases anyway (populating beacon data as the user interacts with the page, for example).

**BFCache-supported ‘unload’-like event**

Another alternative is to introduce (yet) another page lifecycle event, that would be essentially the “unload” event, but supported by the BFCache - that is, its presence would not disable the BFCache, and the browser would execute this callback even on eviction from the BFCache. This was rejected because it would require allowing pages frozen in the BFCache to execute a JavaScript callback, and it would not be possible to restrict what that callback does (so, a callback could do things other than sending a beacon, which is not safe). It also doesn’t allow for other niceties such as resilience against crashes or batching of beacons, and complicates the already sufficiently complicated page lifecycle.


[sendBeacon-api]: https://developer.mozilla.org/en-US/docs/Web/API/Navigator/sendBeacon
[sendBeacon-w3]: https://www.w3.org/TR/beacon/#sec-sendBeacon-method
[ArrayBuffer-api]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/ArrayBuffer
[ArrayBufferView-api]: https://developer.mozilla.org/en-US/docs/Web/API/ArrayBufferView
[Blob-api]: https://developer.mozilla.org/en-US/docs/Web/API/Blob
[FormData-api]: https://developer.mozilla.org/en-US/docs/Web/API/FormData
[URLSearchParams-api]: https://developer.mozilla.org/en-US/docs/Web/API/URLSearchParams
[USVString]: https://www.w3.org/TR/WebIDL-1/#idl-USVString

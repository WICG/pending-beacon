# Stateful JavaScript Page Unload Beacon API


Authors: [Darren Willis](https://github.com/darrenw) - Google

## Stateful Javascript Page Unload Beacon API

This document is an explainer for a system for sending beacons when documents are discarded, that uses a stateful API rather than having developers explicitly send beacons themselves.


## Problem And Motivation

Web developers have a need for ‘beaconing’ - that is, sending a bundle of data to a backend server, without expecting a particular response, ideally at the ‘end’ of a user’s visit to a page. There are currently [four major methods](https://calendar.perfplanet.com/2020/beaconing-in-practice/) of beaconing used around the web:

*   Adding `<img>` tags inside dismissal events.
*   Sending a sync [`XMLHttpRequest`](https://developer.mozilla.org/en-US/docs/Web/API/XMLHttpRequest). Note: doesn’t work as part of dismissal events.
*   The [`Navigator.sendBeacon`][sendBeacon-api] API.
*   Using the [`fetch`](https://developer.mozilla.org/en-US/docs/Web/API/fetch) API with the `keepalive` flag.

(There may be other methods; these are the main ones.)

These methods all suffer from reliability problems, stemming from one core issue: There is not an ideal time in a page’s lifecycle to make the Javascript call to send out the beacon. [`unload`](https://developer.mozilla.org/en-US/docs/Web/API/Window/unload_event) and [`beforeunload`](https://developer.mozilla.org/en-US/docs/Web/API/Window/beforeunload_event) are unreliable (and outright ignored by several major browsers), and [`pagehide`](https://developer.mozilla.org/en-US/docs/Web/API/Window/pagehide_event) and [`visibilitychange`](https://developer.mozilla.org/en-US/docs/Web/API/Document/visibilitychange_event) have [issues](https://github.com/w3c/page-visibility/issues/59) on mobile platforms.

To simplify this issue and make beaconing more reliable, we propose adding a stateful Javascript API where a page can register that it wants a beacon (or beacons) issued when it unloads or the page is hidden. Developers can populate beacon(s) with data as the user uses the page, and the browser ensures beacon(s) are reliably sent at some point in time. This frees developers from worrying about which part of the page lifecycle to send their beacon calls in.


## Goals

Provide a conservatively scoped API, which allows website authors to specify one or more beacons (HTTP requests) that should be sent reliably when the document is being unloaded.


## Requirements

*   The beacon should be sent at or close to document discard time.
    *   For frozen pages that are never unfrozen, this should happen either when the frozen page is removed from memory (BFCache eviction), or after a developer-specified timeout (using the `'pageHideTimeout'` described below)
    *   For browser crashes, forced app closures, etc, the browser should make an effort to send the beacons the next time it is launched (guarantees around privacy and reliability here will be the same as the Reporting API’s crash reporting).
*   The beacon destination URL should be modifiable.
*   The beacon should be cancelable.
*   The beacon should be visible to (and blockable by) extension, to give users control over beacons if they so choose (as they do over current beaconing techniques).


## Design

Our basic idea is to extend the existing Javascript beacon API by adding a stateful version. Rather than a developer calling navigator.sendBeacon(), the developer registers that they would like to send a beacon for this document when it gets discarded, and the browser returns a handle to an object that represents a beacon that the browser promises to send on document discard(whenever that is). The developer can then call methods on this registered beacon handle to populate it with data. Then, at some point later after the user leaves the page, the browser will send the beacon. From the point of view of the developer the exact beacon send time is unknown.


### JavaScript API

 In detail, the proposed design is a new class `PendingBeacon`, constructed like so:


```
beacon = new PendingBeacon(url, options = {});
```

An instance of `PendingBeacon` represents a beacon that will be sent by the browser at some point in the future. The `url` parameter is the same as the existing [`Navigator.sendBeacon`][sendBeacon-api]’s parameter. Note that multiple instances of `PendingBeacon` can be made, so multiple beacons can be sent to multiple endpoints. The `options` parameter would be a dictionary that optionally allows specifying the `'method'` and `'pageHideTimeout'` properties for the beacon (these properties are described below).

Note that calling the PendingBeacon constructor queues the beacon for sending by the browser; even if the result goes out of scope, the beacon will still be sent (unless `deactivate()`-ed beforehand).

The `PendingBeacon` class would support the following methods/properties:

<table>
  <tr>
   <td style="background-color: #efefef"><em>Method/Property Name</em>
   </td>
   <td style="background-color: #efefef"><em>Description</em>
   </td>
  </tr>
  <tr>
   <td><code>deactivate()</code>
   </td>
   <td>Deactivate (cancel) the pending beacon.
   </td>
  </tr>
  <tr>
   <td><code>url</code>
   </td>
   <td>Property reflecting the target endpoint of the pending beacon. Can be reset to point the beacon to a new endpoint.
   </td>
  </tr>
  <tr>
   <td><code>setData(data)</code>
   </td>
   <td>Set the current beacon data. The data argument would take the same types as the <a href="https://developer.mozilla.org/en-US/docs/Web/API/Navigator/sendBeacon">sendBeacon</a> method’s data parameter (that is: “A <code><a href="https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/ArrayBuffer">ArrayBuffer</a></code>, <code><a href="https://developer.mozilla.org/en-US/docs/Web/API/ArrayBufferView">ArrayBufferView</a></code>, <code><a href="https://developer.mozilla.org/en-US/docs/Web/API/Blob">Blob</a></code>, <code><a href="https://developer.mozilla.org/en-US/docs/Web/API/DOMString">DOMString</a></code>, <code><a href="https://developer.mozilla.org/en-US/docs/Web/API/FormData">FormData</a></code>, or <code><a href="https://developer.mozilla.org/en-US/docs/Web/API/URLSearchParams">URLSearchParams</a></code> object containing the data to send.”)
   </td>
  </tr>
  <tr>
   <td><code>sendNow()</code>
   </td>
   <td>Send the current beacon data immediately. Beacons are only ever sent once, so a beacon sent via <code>sendNow()</code> will not be re-sent on document discardor <code>pageHideTimeout</code>. Calling <code>sendNow()</code>on a beacon in any <code>state</code> other than “pending” will throw an exception.
   </td>
  </tr>
  <tr>
   <td><code>method</code>
   </td>
   <td>Accessor property defining the method used to send the beacon. A string matching either “GET” or “POST”. By default, POST is used.
   </td>
  </tr>
  <tr>
   <td><code>pageHideTimeout</code>
   </td>
   <td>Defaults to null. If set, a timeout in milliseconds after the next pagehide event is sent, after which a beacon will be queued for sending, regardless of whether or not the document has been discarded yet. If this is null when the page is hidden, the beacon will be sent on document discard (including eviction from the BFCache). Note that the beacon is not guaranteed to be sent at exactly this many milliseconds after pagehide; bundling/batching of beacons is possible.
   </td>
  </tr>
  <tr>
   <td><code>state</code>
   </td>
   <td>A property holding the current state of the beacon, which may be one of:<ul>

<li>“pending”
<li>“sending”
<li>“sent”
<li>“failed”
<li>“deactivated”

<p>
A beacon starts in the ‘pending’ state. Calling deactivate() on the beacon moves it to the ‘deactivated’ state. The beacon moves to the 'sending’ state as soon as the browser starts sending the beacon, and moves to 'sent' or 'failed' depending on if the beacon send succeeded or failed.</li></ul>

   </td>
  </tr>
</table>

Requests sent by the pending beacon will include cookies (the same as requests from <code>[navigator.sendBeacon][sendBeacon-api]</code>).


### Payload

The payload for the beacon will depend on the method used for sending the beacon. If sent using a POST request, the beacon’s data will be included in the body of the POST request exactly as when <code>[navigator.sendBeacon](https://developer.mozilla.org/en-US/docs/Web/API/Navigator/sendBeacon)</code> is used.

For beacons sent via a GET request, the data will be encoded as query parameters in form application/x-www-form-urlencoded.


### Extensions

Beacons will be sent with the [resource type](https://developer.mozilla.org/en-US/docs/Mozilla/Add-ons/WebExtensions/API/webRequest/ResourceType) of ‘beacon’ (or possibly ‘ping’, as Chromium currently sends beacons with the ‘ping’ resource type). Existing extension APIs that are able to block requests based on their resource types will be able to block these beacons as well.


## Implementation Considerations

This document intentionally leaves out the browser-side implementation details of how beacons will be sent, this section is here merely to note that there are several considerations browser authors may want to keep in mind:

*   Bundling/batching of beacons. Beacons do not need to be sent instantly on document discard, and particularly for mobile devices, batching may improve radio efficiency.
*   Robustness against crashes/forced terminations/network outages.
*   User privacy (see next section).


## Privacy

This design has limited privacy ramifications above the existing beaconing methods - it extends the existing beacon API and makes it more reliable. However, it may break existing means that users have of blocking beaconing - since the browser itself sends beacons ‘behind the scenes’ (so to speak), special support may be needed to allow extension authors to block the sending (or registering) of beacons.

Specifically, beacons will have the following privacy requirements:
*   Beacons must be sent over HTTPS
*   Beacons are only sent over the same network that was active when the beacon was registered (e.g. if the user goes offline and moves to a new network, discard pending beacons)
*   Delete beacons for a site if a user clears site data.
*   Beacons registered in an incognito session do not persist to disk
*   Follow third-party cookie rules for beacons
*   Post-unload beacons are not sent if background sync is disabled for a site.

## Alternatives considered

**DOM-Based API**

A DOM-based API was considered as an alternative to this approach. This API would consist of a new possible ‘beacon’ value for the ‘rel’ attribute on the link tag, which developers could use to indicate a beacon, and then use standard DOM manipulation calls to change the data, cancel the beacon, etc.

The stateful JS API was preferred to avoid beacon concerns intruding into the DOM, and because a ‘DOM-based’ API would still require scripting in many cases anyway (populating beacon data as the user interacts with the page, for example).

**BFCache-supported ‘unload’-like event**

Another alternative is to introduce (yet) another page lifecycle event, that would be essentially the “unload” event, but supported by the BFCache - that is, its presence would not disable the BFCache, and the browser would execute this callback even on eviction from the BFCache. This was rejected because it would require allowing pages frozen in the BFCache to execute a Javascript callback, and it would not be possible to restrict what that callback does (so, a callback could do things other than sending a beacon, which is not safe). It also doesn’t allow for other niceties such as resilience against crashes or batching of beacons, and complicates the already sufficiently complicated page lifecycle.


[sendBeacon-api]: https://developer.mozilla.org/en-US/docs/Web/API/Navigator/sendBeacon

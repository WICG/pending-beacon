# Pending Beacon API

[![Super-Linter](https://github.com/WICG/pending-beacon/actions/workflows/linter.yml/badge.svg)](https://github.com/WICG/pending-beacon/actions/workflows/linter.yml)
[![Spec Prod](https://github.com/WICG/pending-beacon/actions/workflows/auto-publish.yml/badge.svg)](https://github.com/WICG/pending-beacon/actions/workflows/auto-publish.yml)

Authors: [Darren Willis](https://github.com/darrenw), [Fergal Daly](https://github.com/fergald), [Ming-Ying Chung](https://github.com/mingyc) - Google

## What is this?

This repository hosts multiple technical explainers for a system for sending beacons when pages are discarded, rather than requiring developers explicitly send beacons themselves.

This document offers an overview of the system and its explainers.

## Motivation

### What is ‘Beacon’?

Web developers have a need for *‘beaconing’* -
that is, sending a bundle of data to a backend server, **without expecting a particular response**,
ideally at the ‘end’ of a user’s visit to a page.
There are currently
[four major methods](https://calendar.perfplanet.com/2020/beaconing-in-practice/) of beaconing used around the web
(there may be other methods; the followings are the main ones):

* Adding `<img>` tags inside dismissal events.
* Sending a sync [`XMLHttpRequest`] (but it doesn’t work as part of dismissal events).
* Using the [`navigator.sendBeacon()`] API.
* Using the [`fetch()`] API with the `keepalive: true` flag.

### Reliability Problem

The above methods all suffer from reliability problems, stemming from one core issue:
**There is not an ideal time in a page’s lifecycle to make the JavaScript call to send out the beacon.**

* [`unload`](https://developer.mozilla.org/en-US/docs/Web/API/Window/unload_event)
    and [`beforeunload`](https://developer.mozilla.org/en-US/docs/Web/API/Window/beforeunload_event) are unreliable,
    and outright ignored by several major browsers.
* [`pagehide`](https://developer.mozilla.org/en-US/docs/Web/API/Window/pagehide_event)
    and [`visibilitychange`](https://developer.mozilla.org/en-US/docs/Web/API/Document/visibilitychange_event)
    have [issues](https://github.com/w3c/page-visibility/issues/59) on mobile platforms.

## Goal

To simplify the above issues and make beaconing more reliable,
this repository proposes adding a stateful JavaScript API, where a page can register that it wants a beacon issued when the Document is unloaded or hidden.

Developers can populate beacon(s) with data as the user uses the page,
and the browser ensures beacon(s) are reliably sent at some point in time.
This frees developers from worrying about which part of the page lifecycle to send their beacon calls in.

## Requirements

The followings are critical:

1. Support a reliable mechanism for delaying operation until page is unloaded, or evicted from bfcached.
2. Behave like a keepalive fetch request when 1's mechanism triggers.
3. Allow pending requests to be updated to reduce network usage.
4. Allow to specify a duration to accelerate beacon sending after page is bfcached.

The followings are good-to-have:

1. When browser crashes, app is forced to close, etc, the browser should make an effort to send the beacons the next time it is launched.
2. The beacon data, including URL and body, should be modifiable.

## JavaScript API

* [**fetchLater() API**](docs/fetch-later-api.md): The latest API proposal, currently under specification.

Previous proposals:

* [DEPRECATED] [fetch() with PendingRequest API](docs/fetch-with-pending-request-api.md): The transitional API proposal and discussions happened between PendingBeacon API and fetchLater API.
* [DEPRECATED] [PendingBeacon API](docs/pending-beacon-api.md): The initial experimental API, available as Chrome Origin Trial from M107 to M115.

## Specification

* Deferred fetching - whatwg/fetch: [PR](https://github.com/whatwg/fetch/pull/1647), [Spec Preview](https://whatpr.org/fetch/1647.html#dom-global-fetch-later)
* Reserve/free quota for fetchLater - whatwg/html: [PR](https://github.com/whatwg/html/pull/10903)

## Alternatives Considered

See [Alternative Approaches](docs/alternative-approaches.md).

## Discussions

* [WebKit Standards Positions](https://github.com/WebKit/standards-positions/issues/85)
* [Mozilla Standards Positions](https://github.com/mozilla/standards-positions/issues/703)
* [TAG Design Review](https://github.com/w3ctag/design-reviews/issues/887)
* Yours - [Open an issue](https://github.com/WICG/pending-beacon/issues/new)

[`XMLHttpRequest`]: https://developer.mozilla.org/en-US/docs/Web/API/XMLHttpRequest
[`navigator.sendBeacon()`]: https://developer.mozilla.org/en-US/docs/Web/API/Navigator/sendBeacon
[`fetch()`]: https://developer.mozilla.org/en-US/docs/Web/API/fetch

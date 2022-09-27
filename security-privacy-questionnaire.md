# [Self-Review Questionnaire: Security and Privacy][self-review]

This questionnaire covers the Page Unload Beacon API [explainer], based on the [W3C TAG Self-Review Questionnaire: Security and Privacy][self-review].

1. What information does this feature expose, and for what purposes?
     > The API intends to provide a reliable way for a document to send data to the target URL on document discard, or sometimes later after the document entering bfcache (becoming non-"fully active"), by making the user agent process the "send" requests queued by calling the API.
     > The user agent should not expose the queued requests to new network providers after users navigating away from the document where the requests were queued.
     > Note that the API only sends requests in CORS mode with Same-Origin credentials mode.

   1. What information does your spec expose to the first party that the first party cannot currently easily determine.
      > No extra information exposed.

   2. What information does your spec expose to third parties that third parties cannot currently easily determine.
      > No extra information exposed.
      > Note that the API only sends requests in CORS mode with Same-Origin credentials mode.

   3. What potentially identifying information does your spec expose to the first party that the first party can already access (i.e., what identifying information does your spec duplicate or mirror).
      > The API spec does not directly deal with potentially identifying information. But the API users can learn when a document is discarded or put into bfcache (becoming non-"fully active"), which is already available by using the corresponding event handlers.

   4. What potentially identifying information does your spec expose to third parties that third parties can already access.
      > Same as above.

2. Do features in your specification expose the minimum amount of information necessary to enable their intended uses?

     > Yes. No information is directly exposed by the API itself.

3. Do the features in your specification expose personal information, personally-identifiable information (PII), or information derived from either?

     > No. The API itself does not expose such information. However, the API users can collect such information and send them via the API.

4. How do the features in your specification deal with sensitive information?

     > No. The API itself does not expose such information. However, the API users can collect such information and send them via the API.

5. Do the features in your specification introduce new state for an origin that persists across browsing sessions?

     > Not across browsing sessions. But the API provides a way to queue HTTP requests on a document which will be send by the user agent sometimes later, but before the document is discarded/browsing session ends.
     > Users can disallow the "sending on document discard" behavior by disabling BackgroundSync for an origin.

6. Do the features in your specification expose information about the underlying platform to origins?

     > No. The API itself does not expose such information. However, the API users can collect such information and send them via the API.

7. Does this specification allow an origin to send data to the underlying platform?

     > No.

8. Do features in this specification enable access to device sensors?

     > No.

9. Do features in this specification enable new script execution/loading mechanisms?

     > No.

10. Do features in this specification allow an origin to access other devices?

     > No. The users can use API to send HTTP requests to a target URL which might be other devices. But the responses are discarded by user agent.

11. Do features in this specification allow an origin some measure of control over a user agent's native UI?

     > No.

12. What temporary identifiers do the features in this specification create or expose to the web?

     > No.

13. How does this specification distinguish between behavior in first-party and third-party contexts?

     > Both 1st party and 3rd party can use the API.
     > But the API only sends requests in CORS mode with Same-Origin credentials mode.

14. How do the features in this specification work in the context of a browser’s Private Browsing or Incognito mode?

     > No difference.

15. Does this specification have both "Security Considerations" and "Privacy Considerations" sections?

     > Yes.

16. Do features in your specification enable origins to downgrade default security protections?

     > No.

17. How does your feature handle non-"fully active" documents?

     > The API provides some mechanism for users to specify when to send the queued requests after the document becomes non-"fully active".
     > The data of the queued requests should only come from the API calls when the document is still active. If the document transits from non-"fully active" to fully active again, the API can continue to accumulate more data from the API calls in the document.
     > If the network provider changes after the document becomes non-"fully active", the user agent should not expose the queued requests to the new network provider.
     > The user agent sends out all queued requests if a non-"fully active" document gets unloaded (discard).

18. What should this questionnaire have asked?

     > How long can a request be queued on a document by this feature?

[self-review]: https://w3ctag.github.io/security-questionnaire/
[explainer]: https://github.com/WICG/unload-beacon/blob/main/README.md

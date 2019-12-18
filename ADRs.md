

## Design Decisions

### Long Polling

A primary concern in using http feeds is latency with client side polling.

Long polling provides near-zero latency, but has higher demands on the server side, as the requests must be kept open.
This should not be a problem for few clients (< 100) and should be feasible for [many parallel clients](https://en.wikipedia.org/wiki/C10k_problem) with reactive implementations.


### Offset Querying

We use dynamic offset querying of the next feed items using the position as offset.

Discussed alternatives:

* Static linked pages, but higher traffic while polling last page and to limited for filtering.

### Plain JSON

We use plain `application/json` as media type.

Discussed alternatives:

- [JSON:API](https://jsonapi.org/), reasonable, but very nested and complex for embedded data (the `included` array). Special `application/vnd.api+json` media type.
- [HAL](http://stateless.co/hal_specification.html), but to limited and not widly supported.
- [JSON-LD](https://json-ld.org/), but not widly adopted.
- [Collection+JSON](http://amundsen.com/media-types/collection/), but unnecessary complex nesting.

### Server defined limit

The page limit is defined by server and cannot be set by the client.

This this is simple, protects the server from to large data sets and DoS, enables caching and enables the server to choose optimized storage systems.

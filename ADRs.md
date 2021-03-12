# Design Decisions

## Long Polling

Long polling significantly simplifies the client algorithm as the client does not need to implement different handling if there are further items in the feed available. There is no sleep step involved in normal processing. Long polling minimizes latency when new items arrive in the feed.

On the downside, long polling has higher demands on the server side, as the requests connections must be kept open for some seconds.
This should not be a problem for a few clients (< 100) and should be feasible for [many parallel clients](https://en.wikipedia.org/wiki/C10k_problem) with reactive implementations.

![Long Polling](https://i.ibb.co/x8jmrrM/Long-polling-1.png)

In a simple implementation, the server queries the database for new items in a short interval (such as 50 millis) until there are new items or a timeout occurs (> 1 seconds, < 10 seconds recommended).


## Offset Querying

We use dynamic offset querying of the next feed items using the position as offset.

Discussed alternatives:

* Static linked pages, but higher traffic while polling last page and to limited for filtering.
* Use `Last-Event-ID` (as specified by server-sent events). Consequence: Position for ID must be retrievable, when entries get deleted. 

## Next link at every item

Every feed item contains a next link to fetch subsequent items.

While it would be sufficient (and feeling more natural) to have a next link on page level, it makes each feed item self-contained.
In case of processing or appliction error in a subsequent item on the same page, the client can make the next fetch starting from the last successfully processed item's next link.
This significantly simplifies exception and retry handling on the client side.


## Plain JSON

We use plain `application/json` as media type and return a simple array of items.

Discussed alternatives:

- [JSON:API](https://jsonapi.org/), reasonable, but very nested and complex for embedded data (the `included` array). Special `application/vnd.api+json` media type.
- [HAL](http://stateless.co/hal_specification.html), but too limited and not widly supported.
- [JSON-LD](https://json-ld.org/), but not widly adopted.
- [Collection+JSON](http://amundsen.com/media-types/collection/), but unnecessary complex nesting.


## Server defined limit

The page limit is defined by server and should be set sized reasonable based on the payload size.
The actual page limit may vary for different pages.

This is simple, protects the server from too large data loads and DoS, enables caching and enables the server to choose optimized storage systems.

Feed endpoints _may_ choose to support a `limit` query parameter, e.g. for low bandwidth clients.
The server _may_ ignore or override the limit. The server _should_ always bound the upper limit.

## No home document

We decided against a home document (such as [JSON Home](https://mnot.github.io/I-D/json-home/)).

REST feeds is designed to be as simple as possible.

The feed starts with the plain feed URL and the client just needs to follow next links.

When the feed endpoint changes, the old endpoint _should_ return a 302 redirect to the new endpoint.

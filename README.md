# rest-feeds

Asynchronous data replication and event streaming with plain REST/HTTP.

## 1-Minute Overview

A REST feed provides access to resources in a _chronological sequence of changes_.

```
GET /movies
Accept: application/json

200 OK
Content-Type: application/cloudevents-batch+json
[{
  "specversion" : "1.0",
  "type" : "org.example.movie",
  "source" : "/movies",
  "id": "11b592ae-490f-4c07-a174-04db33c2df70",
  "time": "2019-12-16T08:41:519Z",
  "subject": "/movies/18",
  "data": {
    "original_title":"The Fifth Element",
    "popularity":26.163
  }
},{
  "specversion" : "1.0",
  "type" : "org.example.movie",
  "source" : "/movies",
  "id": "64e11a7a-0e40-426c-8d81-259d6f6ab74e",
  "time": "2019-12-16T09:12:421Z",
  "subject": "/movies/12",
  "data": {
    "original_title":"Finding Nemo",
    "popularity":23.675
  }
}]
```

The number of items returned by the server is limited, e. g. to 1000 items per request.
Repeat the request with the id that has been processed as `lastEventId` query parameter. 
Note that this is an example of an [Aggregate Feed](#aggregate-feed) where resources can be updated and deleted:

```
GET /movies?lastEventId=64e11a7a-0e40-426c-8d81-259d6f6ab74e
Accept: application/json

200 OK
Content-Type: application/cloudevents-batch+json
[{
  "specversion" : "1.0",
  "type" : "org.example.movie",
  "source" : "/movies",
  "id": "756e21c9-4ebd-4354-8f7d-85cd7d2bc4ec",
  "time": "2019-12-17T11:09:122Z",
  "subject": "/movies/18",
  "data": {
    "original_title":"The Fifth Element",
    "popularity":27.011
  }
},{
  "specversion" : "1.0",
  "type" : "org.example.movie",
  "source" : "/movies",
  "id": "e510d24e-bf06-4f6a-b6db-5744f6ff2591",
  "subject": "/movies/12",
  "time": "2019-12-18T17:00:786Z",
  "method": "DELETE"
}]
```

The server implements [long polling](#feed-endpoint).
When there are no newer items, the server holds the connection open until new data or a timeout arrives. 

Example for no new data after 5 seconds:

```
GET /movies?lastEventId=e510d24e-bf06-4f6a-b6db-5744f6ff2591
Accept: application/json

200 OK
Content-Type: application/cloudevents-batch+json
[]
```

The client continues polling this link until new items are received.

## REST Feeds

REST feeds provide access to resources in a _chronological sequence of changes_ using plain HTTP(S).

A REST feed complies to these principles:

* HTTP(S) as transfer protocol
* Clients poll the feed endpoint for new items
* Paged results with links to further items
* Content-Negotiation, with `application/cloudevents-batch+json` as default
* Feed items are immutable.
* Items are always appended.

REST feeds enable asynchronously decoupled systems without shared infrastructure.

REST feeds can be used for data replication (_aggregate feeds_) and event streaming (_event feeds_).

### Aggregate Feeds

_Aggregate feeds_ are used to share aggregates (aka master data, domain objects) with other systems for data replication.

Every update to the resource leads to a new entry of the full current state in the feed.

An aggregate feeds must contain every resource (identified through its `subject`) at least once. 

### Event Feeds

_Event feeds_ are used to publish [domain events](https://www.innoq.com/en/blog/domain-events-versus-event-sourcing/#domainevents) that have happened in a domain.
Event feeds often contain different entry `type`s. 

Events may be deleted once they are outdated or useless.

## Specification

### Feed Endpoint

The feed endpoint _must_ return items in a strictly ascending order of addition to the feed.

The feed endpoint _must_ support fetching items without any query parameters, starting with the first items.

The feed endpoint _may_ choose to limit the number of items returned in a response to a subset of the whole set available. 

The feed endpoint _must_ must support the `lastEventId` query parameter.
It must only returns items after this event id.

The feed endpoint _must_ implement [long polling](https://en.wikipedia.org/wiki/Push_technology#Long_polling).
If the server has no newer items, the server holds the request open until new items arrive and then immediately sends the response.

The server _should_ send an empty response if no new item arrived after N seconds (5 seconds recommended).

### Feed Clients

The initially stored link is the feed endpoint URL without query parameters.

Pseudocode:

```python
link = "https://feed.example.com/movies"
while true:
  try:
    response = GET link
    for item in response:
      process item
      link = link + setQueryParameter "lastEventId" to value item.id
  except:
    wait N seconds
```

The client _must_ persist the `id` of the last processed item.

The client's item handling _must_ be idempotent (_at-least-once_ delivery semantic). 
The `id` _may_ be used for idempotency checks.

The client _must_ implement an exception handler that delays the next request, to protect the server in case of connection or processing errors.

### Model

The response contains an array of _items_.
The response must comply with the [CloudEvents Specification](https://github.com/cloudevents/spec).

Field    | Type   | Mandatory | Description
---      | ---    | ---       | ---
`specversion`     | String | Mandatory | The currently supported CloudEvents specification version.
`id`     | String | Mandatory | A unique value (such as a UUID) for this item. It can be used to implement deduplication/idempotency handling in downstream systems.
`type`   | String | Mandatory | The type of the item. May be used to specify and deserialize the payload. A feed may contain different item types, especially if it is an event feed. It SHOULD be prefixed with a reverse-DNS name..
`subject` | String | Optional | A [URI](https://en.wikipedia.org/wiki/Uniform_Resource_Identifier) to the resource, the feed item refers to. It doesn't have to be unique within the feed. This should include a business key, such as an order number.
`method` | String | Optional | The HTTP equivalent method type that the feed item performs on the `subject`. `PUT` indicates that the _resource_ was created or updated. `DELETE` indicates that the  _subject_ was deleted. Defaults to `PUT`.
`time` | String | Mandatory | The item addition timestamp. ISO 8601 UTC date and time format.
`data`   | Object | Optional  | The payload of the item in JSON. May be missing, e.g. when the method was `DELETE`.

Further metadata may be added, e.g. for traceability.


### Content Negotiation

Every consumer and provider _must_ support the media type `application/cloudevents-batch+json` as defined by the [CloudEvents JSON Batch Format](https://github.com/cloudevents/spec/blob/v1.0.1/json-format.md#4-json-batch-format).
It is the default and used when the `Accept` header is missing, defined as plain `application/json` or not supported.

Further media types may be used, when supported by both, client and server:

* `text/html` to render feed in browser
* `application/atom+xml` to support feed readers
* `text/event-stream` for server-sent events
* `application/x-ndjson` to support streaming
* `multipart/*` to have a more HTTP native representation
* `application/x-protobuf` to minimize traffic
* any other
 

### Authentication

Feed endpoints _may_ be protected with [HTTP authentication](https://developer.mozilla.org/en-US/docs/Web/HTTP/Authentication). 

The most common authentication schemes are [Basic](https://tools.ietf.org/html/rfc7617) and [Bearer](https://tools.ietf.org/html/rfc6750).

The server _may_ filter feed items based on the principal.
When filtering is applied, [caching](#caching) may be unfeasible.


### Compaction

When feed items include the full current state of the resource, older feed items for the same resource may be obsolete. 
Items _may_ be deleted from the feed when another item was added to the feed with the same `resource` URI.

It is good practice to keep the feed small to enable a quick synchronization of new clients.

The server _must_ handle next links, when the requested item has been deleted by returning the next higher items.

### Deletion

When a resource was deleted, the server _must_ append a `DELETE` item with the `resource` URI to delete.

Clients _must_ delete this resource or otherwise handle the removal.

```
{
  "specversion" : "1.0",
  "type" : "org.example.movie",
  "source" : "/movies",
  "id": "8a22af5e-d9ea-4e7f-a907-b5e8687800fd",
  "time": "2019-12-17T07:07:777Z",
  "subject": "/movies/777777",
  "method": "DELETE"
}
```

The server _should_ start a [compaction](#compaction) run afterwards to delete previous items for the same resource.

### Caching

Feed endpoints _may_ set [appropriate](https://devcenter.heroku.com/articles/increasing-application-performance-with-http-cache-headers) response headers, such as `Cache-Control: public, max-age=31536000`, when a page is full and will not be modified anymore.

## Code & Libraries

- [restfeed-server-java](https://github.com/rest-feeds/restfeed-server-java) 
- [restfeed-server-spring](https://github.com/rest-feeds/restfeed-server-spring) 
- [restfeed-server-spring-example](https://github.com/rest-feeds/restfeed-server-spring-example) 
- [restfeed-client-java](https://github.com/rest-feeds/restfeed-client-java)
- [restfeed-client-spring](https://github.com/rest-feeds/restfeed-client-spring)



## More Information

- [FAQ](FAQ.md)
- [Design decisions](ADRs.md)

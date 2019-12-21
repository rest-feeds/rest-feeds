# REST Feeds

Publish data and events using HTTP and REST.

This site describes the [concept](#rest-feeds-1) of REST feeds and proposes a [data model](#model).


## Example

A REST feed provides access to resources in a chronological sequence of changes.

```http
GET /movies
Accept: application/json

200 OK
Content-Type: application/json
[{
  "id": "11b592ae-490f-4c07-a174-04db33c2df70",
  "next": "/movies?offset=126",
  "type": "application/vnd.org.themoviedb.movie",
  "uri": "/movies/18",
  "timestamp": "2019-12-16T08:41:519Z",
  "data": {
    "original_title":"The Fifth Element",
    "popularity":26.163
  }
},{
  "id": "64e11a7a-0e40-426c-8d81-259d6f6ab74e",
  "next": "/movies?offset=127",
  "type": "application/vnd.org.themoviedb.movie",
  "uri": "/movies/12",
  "timestamp": "2019-12-16T09:12:421Z",
  "data": {
    "original_title":"Finding Nemo",
    "popularity":23.675
  }
}]
```

Repeat the request with the `next` link of the last processed item. 
Note that this is an example of a [data feed](#data-feed) where resources can be updated and deleted:

```http
GET /movies?offset=127
Accept: application/json

200 OK
Content-Type: application/json
[{
  "id": "756e21c9-4ebd-4354-8f7d-85cd7d2bc4ec",
  "next": "/movies?offset=128",
  "type": "application/vnd.org.themoviedb.movie",
  "uri": "/movies/18",
  "timestamp": "2019-12-17T11:09:122Z",
  "data": {
    "original_title":"The Fifth Element",
    "popularity":27.011
  }
},{
  "id": "e510d24e-bf06-4f6a-b6db-5744f6ff2591",
  "next": "/movies?offset=129",
  "type": "application/vnd.org.themoviedb.movie",
  "uri": "/movies/12",
  "method": "DELETE",
  "timestamp": "2019-12-18T17:00:786Z"
}]
```

The server implements [long polling](#feed-endpoint).
When there are no newer items, the server holds the connection open until new data or a timeout arrives. 

Example for no new data after x seconds:

```http
GET /movies?offset=129
Accept: application/json

200 OK
Content-Type: application/json
[]
```

The client continues polling this link until new items are received.

## Why

Independent information systems, such as microservices or [Self-contained Systems](https://scs-architecture.org/), communicate with other systems to publish data or events. 

Often this integration is made with technologies, like:

- Synchronous API calls (REST-APIs, SOAP Web Services, BAPI, ...)
- Message Brokers (Kafka, JMS, MQ, SQS, ...)
- Database Integration

All these integration strategies lead to tight coupling with technical and organizational dependencies and all its problems. 
Think of ownership, availability, resilience, changeability, and firewalls.

REST feeds are an alternative for data publishing between systems. 
They use the existing HTTP infrastructure and communication patterns.

## REST Feeds

REST feeds provide access to resources in a chronological sequence of changes.
Feed items are immutable.
New items are always appended.
Clients poll the feed endpoint for new items.

A REST feed complies to these principles:

* HTTP(S) as transfer protocol
* Polling-based, client initiated GET requests only
* Paged collections with link to the next page
* Content-Negotiation, with `application/json` as default

REST feeds enable asynchronously decoulped systems without shared infrastructure.

### Data Feeds

_Data feeds_ are used to share resources (master data, domain objects, aggregates) with other systems for data replication. 
Usually a data feed contains only entries of one resource `type`.

Typical examples: 

- _Customers_
- _Products_
- _Stock_

Every update to the resource leads to a new entry of the full current state in the feed.

A data feed must contain every resource (identified through its `uri`) at least once. 

### Event Feeds

_Event feeds_ are used to publish [domain events](https://www.innoq.com/en/blog/domain-events-versus-event-sourcing/#domainevents) that have happened in a domain.
Event feeds often contain different entry `type`s. 

Typical examples: 

- _PaymentProcessed_
- _OrderShipped_
- _AccountLocked_

Events may be deleted once they are outdated or useless.


## Feed Endpoint

The feed endpoint _must_ return items in strictly ascending order of addition to the feed.

The feed endpoint _must_ support fetching items without any query parameters, starting with the first items.

The feed endpoint _may_ choose to limit the number of items returned in a response to a subset of the whole set available. 

The feed endpoint _must_ add a `next` link to every returned item.
The `next` Link must return subsequent items.

The feed endpoint _must_ implement [long polling](https://en.wikipedia.org/wiki/Push_technology#Long_polling).
If the server has no newer items, the server holds the request open until a new item arrive and then immediatelly send the response.

The server _should_ send an empty response, if no new item arrived after N seconds (5 seconds recommended).

## Client Behaviour

The initial stored link is the feed endpoint URL without query parameters.

Pseudocode:

```python
link = "https://feed.example.com/movies"
while true:
  try:
    response = GET link
    for item in response:
      process item
      link = item.next
  except:
    wait N seconds
```

The client _must_ persist the `next` link of the last processed item.

The client's item handling _must_ be idempotent (_at-least-once_ delivery semantic). The `id` _may_ be used for idempotency checks.

The client _must_ implement an exception handler that delays the next request, to protect the server in case of connection or processing errors.

## Model

The response contains an array of _items_.

Field    | Type   | Mandatory | Description
---      | ---    | ---       | ---
`id`     | String | Mandatory | A unique value (such as a UUID) for this item. Can be used to implement deduplication/idempotency handling in downstream systems.
`next`   | String | Mandatory | A link to subsequent items. Fetching the link returns a (paged) collection with subsequent items (without the current item).
`type`   | String | Mandatory | The type of the item. Usually used to deserialize the payload. A feed may contain different item types, especially if it is an event feed. It is recommended to use a namespaced [media type](https://en.wikipedia.org/wiki/Media_type).
`uri`    | String | Optional | A [URI](https://en.wikipedia.org/wiki/Uniform_Resource_Identifier) to this resource. Doesn't have to be unique within the feed.
`method` | String | Optional | `PUT` indicates that the resource for the _uri_ was created or updated. `DELETE` indicates that the resource for the _uri_ was deleted. Defaults to `PUT`.
`timestamp` | String | Mandatory | The item addition timestamp. ISO 8601 UTC date and time format.
`data`   | Object | Optional  | The payload of the item. May be missing, e.g. when the method was `DELETE`.

Further meta data may be added, e.g. for traceability.


## Content Negotiation

Every consumer and provider _must_ support the media type `application/json`.
It is the default and used, when the `Accept` header is missing or not supported.

Further media types may be used, when supported by both, client and server:

* `text/html` to render feed in browser
* `application/atom+xml` to support feed readers
* `text/event-stream` for server-side events
* `application/x-ndjson` to support streaming
* `multipart/*` to have a more HTTP native representation
* `application/x-protobuf` to minimize traffic
* any other


## Authentication

Feed endpoints _may_ be protected with [HTTP authentication](https://developer.mozilla.org/en-US/docs/Web/HTTP/Authentication). 

The most common authentication schemes are [Basic](https://tools.ietf.org/html/rfc7617) and [Bearer](https://tools.ietf.org/html/rfc6750).

The server _may_ filter feed items based on the principal.
When filtering is applied, [caching](#caching) may be unfeasible.


## Compaction

_Compaction is usually only relevant in [data feeds](#data-feeds)._

Items _may_ be deleted from the feed, when another item was added to the feed with the same `uri`.

The server _must_ handle next links, when the requested item has been deleting by returning the next higher items.

## Deletion

_Deletion is usually only relevant in [data feeds](#data-feeds)._

When a resource was deleted, the server _must_ append a `DELETE` item with the same `uri` as resource to delete.

Clients _must_ delete this resource or otherwise handle the removal.

```
{
  "id": "8a22af5e-d9ea-4e7f-a907-b5e8687800fd",
  "next": "/movies?offset=321",
  "type": "application/vnd.org.themoviedb.movies",
  "uri": "/movies/777777",
  "method": "DELETE",
  "timestamp": "2019-12-17T07:07:777Z"
}
```

The server _should_ start a [compaction](#compaction) run afterwards to delete previous items for the same URI.

## Caching

Feed endpoints _may_ set an [appropriate](https://devcenter.heroku.com/articles/increasing-application-performance-with-http-cache-headers) response headers, such as `Cache-Control: public, max-age=31536000` , when a page is full and will not be modified any more.


## More information

- [FAQ](FAQ.md)
- [Design decisions](ADRs.md)

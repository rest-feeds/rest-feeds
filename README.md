# REST Feeds

Publish data and events using HTTP and REST.

This site describes the concept of REST Feeds and proposes a data model.



## Example

A REST feed provide access to resources in a chronological sequence of changes.

```http
GET /orders
Accept: application/json

200 OK
Content-Type: application/json
[{
  "id": "11b592ae-490f-4c07-a174-04db33c2df70",
  "next": "/orders?offset=126",
  "type": "application/vnd.com.example.order",
  "uri": "/orders/123456",
  "method": "PUT",
  "timestamp": "2019-12-16T08:41:59Z",
  "data": {
    "foo": "bar"
  }
},{
  "id": "64e11a7a-0e40-426c-8d81-259d6f6ab74e",
  "next": "/orders?offset=127",
  "type": "application/vnd.com.example.order",
  "uri": "/orders/777777",
  "method": "PUT",
  "timestamp": "2019-12-16T09:12:421Z",
  "data": {
    "foo": "baz"
  }
}]
```

The number of items in the response may be limited.
Repeat the request with the `next` link of the last processed item. 

```http
GET /orders?offset=127
Accept: application/json

200 OK
Content-Type: application/json
[]
```

The server uses [long polling](Long-Polling).
When there are no newer items, the server holds the connection open  until new data or a timeout arrives.

## Why

Independent information systems, such as microservices or [Self-contained Systems](https://scs-architecture.org/), communicate with other systems to publish data or events. 
For example, when an order was made in an online shop, the order must be forwarded to the fulfillment system to ship the order.

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
* Polling-based, client initiated GET requests, only
* Long Polling, the client gets updates immediatelly, without delay
* Hypermedia link, to the next page
* Content-Negotiation, with `application/json` as default

REST feeds enable asynchronously decoulped systems without shared infrastructure.

## Data Feeds and Event Feeds

REST feeds supports both, data feeds and event feeds.

### Data Feeds

_Data feeds_ are used to share domain objects (aggregates) with other systems. 
Downstream systems can replicate the data for reference and lookups.
Usually a data feed contains only entries of one `type`.

Typical examples: 

- Customers
- Products
- Stock

Every update to the domain object leads to a new entry of the full current state in the feed.

A data feed must contain every domain object (identified through its `uri`) at least once. 
Outdated items may be compacted.

### Event Feeds

_Event feeds_ are used to publish [domain events](https://www.innoq.com/en/blog/domain-events-versus-event-sourcing/#domainevents) that have happened in a domain.
Downstream systems may trigger processes based on these events.
Event feeds often contain different entry `type`s. 

Typical examples: 

- _PaymentProcessed_
- _OrderShipped_
- _AccountLocked_

Usually there is no compaction in event feeds, but events may be deleted once they are outdated or useless.


## Feed Endpoint

The feed endpoint _must_ support fetching items without any query parameters, starting with the first items in ascending order.

The feed endpoint _may_ choose to limit the number of items returned in a response to a subset of the whole set available. 
The limit is defined by the server.

The feed endpoint _must_ add a `next` link to every returned item.
The `next` Link must return all subsequent items.



## Client Behaviour

The initial stored link is the feed endpoint URL without query parameters.

Pseudocode:

```python
link = "https://feed.example.com/orders"
while true
  response = GET link
  for item in response:
    process item
    link = item.next
```

The client has the responsibility to persist the `next` link of the last processed item.
The client`s item handling must be idempotent, as the processing of further items or the persistence operation may fail. 


## Model

The response contains two top level elements: A `links` object and an `items` array.

### Links

Link    | Description
---     | ---
`self`  | The link of the current page. Poll this link periodically with some delay, when no `next` link is set.
`next`  | The link indicates, that more data is available. Call this link immediatelly when all items have been processed.

### Items

Field    | Type   | Optional  | Description
---      | ---    | ---       | ---
`position` | Number :warning: | Mandatory | The `position` must be strictly monotonic increasing. Values may be skipped. Used to query feed items.
`meta`   | Object | Mandatory | Metadata about the item.
`links`  | Object | Optional  | Links to this item.
`data`   | Object | Optional  | The payload of the item. May be missing, e.g. when the item represents a tombstone.


### Meta

Field    | Type   | Optional  | Description
---      | ---    | ---       | ---
`type`   | String | Mandatory | The type of the item. A feed may contain different item types, especially if it is an event feed. It is recommended to use a namespace, such as a hierarchical naming pattern.
`id` :warning:    | String | Mandatory | An identifier for this resource. In data feeds this is usually a business object id. In event feeds this is an event id. 
`operation` | String | Optional | `put` indicates that the resource for the _id_ was created or updated. `delete` indicates that the resource for the _id_ was deleted. Defaults to `put`, if omitted. 
`created` | String | Mandatory | The timestamp of the item addition to the feed. ISO 8601 UTC date and time format.
`idempotencyKey` | String | Optional | A unique value (such as a UUID) for this item to support idempotency handling in downstream systems.

Further meta data may be added, e.g. for traceability.

## Pages and Polling

:warning: TODO 



## Content Negotiation

Every consumer and provider _must_ support the media type `application/json`.
It is the default and used, when the `Accept` header is missing or not supported.

Further media types may be used, when supported by both, client and server:

* `text/html` to render feed in browser
* `application/atom+xml` to support feed readers
* `application/x-ndjson` to support streaming
* `multipart/*` to have a more HTTP native representation
* `application/x-protobuf` to minimize traffic
* `TODO` AVRO to minimize traffic
* any other



## Tombstone

:warning: TODO 


## Compaction

Items may be deleted from the feed, when another item was added to the feed with the same `id`.

This leads to sparse `position`s.
The server _must_ handle offset requests, when the requested position has been deleting by returning the next higher positions.

## Authentication

Feed endpoints _may_ be protected with [HTTP authentication](https://developer.mozilla.org/en-US/docs/Web/HTTP/Authentication). 

The most common authentication schemes are [Basic](https://tools.ietf.org/html/rfc7617) and [Bearer](https://tools.ietf.org/html/rfc6750).

The server _may_ filter feed items based on the principal.
When filtering is applied, caching may be unfeasible.


## Filter

Feed endpoints _may_ support the `filter` query parameter to return a subset of items, following the recommendations in [JSON:API filtering](https://jsonapi.org/recommendations/#filtering).

Example:

```
GET /orders?offset=123&filter[type]=com.example.order&filter[id]=123456,123457
```

When filtering is applied, caching may be unfeasible.

## Caching Strategies

https://wiki.innoq.com/display/CC/FINT+2.+Feed+Basics#FINT2.FeedBasics-2.5.Cachingstrategy

## Long Polling

The server _may_ support long-polling, especially when latency is a major issue.

:warning: TODO 


## FAQ


### Why not RSS/ATOM?

[RSS](http://www.rssboard.org/rss-specification) and [ATOM](https://tools.ietf.org/html/rfc4287) are the archetypes for REST feeds.
Their main concepts are adopted.

They where created to publish news articles and podcasts in the web.
Their model with attributes, such as `author`, `title`, or `summary` simply doesn`t fit well for data and event feeds. Pagination is not specified.

Plus, they enforce the usage of XML.
Nowadays, it is rather common to use JSON as an exchange data format.
Or event better, to let the client decide witch format to use.
The adoption [jsonfeed.org](https://jsonfeed.org/) was never widly used and is still focusing in articles.



## Design Decisions

### Offset Querying

We use dynamic offset querying of the next feed items using the position as offset.

Discussed alternatives:

* Static linked pages, but higher traffic while polling last page and to limited for filtering.

### Plain JSON

We use plain `application/json` as media type.

Discussed alternatives:

- [JSON:API](https://jsonapi.org/) has sensible defaults to representante collection data in JSON.
Many concepts are adopted. But shortcomings, such as the `included` array for inline data and the `application/vnd.api+json` media type.
- [HAL](http://stateless.co/hal_specification.html), but to limited and not widly supported.
- [JSON-LD](https://json-ld.org/), but not widly adopted.
- [Collection+JSON](http://amundsen.com/media-types/collection/), but unnecessary complex nesting.

### Server defined limit

The page limit is defined by server and cannot be set by the client.

This this is simple, protects the server from to large data sets and DoS, enables caching and enables the server to choose optimized storage systems.

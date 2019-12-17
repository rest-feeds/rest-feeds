# REST Feeds

Publish data and events using HTTP and REST.

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
Clients periodicly poll the feed endpoint for updates.
The server sends paged collections of data and presents a _next_ link.

A REST feed complies to these principles:

* HTTP(S) as transfer protocol
* Polling-based, GET requests only
* Linked Pages, with hypermedia link to the next page, if available
* Content-Negotiation, with JSON as default

REST feeds enable asynchronously decoulped systems without shared infrastructure.

## Example

![rest-feeds](rest-feeds.svg)

:warning: Fit svg to example

```
GET /orders?offset=123
Accept: application/json
```

```json
200 OK
Content-Type: application/json
{
  "links": {
    "self": "/orders?offset=123",
    "next": "/orders?offset=126"
  },
  "items": [
    {
      "position": 124,
      "meta": {
        "type": "com.example.order",
        "id": "0a26677b-cb72-499c-8d6e-f69af399c724",
        "key": "123456",
        "operation": "put",
        "created": "2019-12-16T08:41:59Z"
      },
      "data": {
        "foo": "bar"
      }
    },
    {
      "position": 126,
      "meta": {
        "type": "com.example.order",
        "id": "f5b5d2f6-433a-4d40-9ee2-f06488b98861",
        "key": "777777",
        "operation": "put",
        "created": "2019-12-16T09:12:421Z"
      },
      "data": {
        "foo": "baz"
      }
    }
  ]
}
```

## Data Feeds and Event Feeds

REST feeds supports both, data feeds and event feeds.

_Data feeds_ are used to share business objects with other systems. 
Downstream systems can replicate the data for reference and lookups.
Usually a data feed contains only entries of one `type`.
Examples: customers, products, stock.
Every update to the business object leads to an entry of the full current state in the feed.
The feed must contain every business object (identified through its `id` :warning:) at least once.

_Event feeds_ are used to publish [domain events](https://www.innoq.com/en/blog/domain-events-versus-event-sourcing/#domainevents) that have happened in a domain.
Downstream systems can trigger processes.
Event feeds often contain different entry `type`s. 
Examples: `PaymentProcessed`, `OrderShipped`, `AccountLocked`.
Every event has a unique `id`.
Usually there is no compaction, but the feed may cut off once the domain events are useless.


## Feed Endpoint

The feed endpoint _must_ support fetching `items` without any query parameters, starting with the first items in ascending order.

The feed endpoint _may_ choose to limit the number of items returned in a response to a subset of the whole set available. 
The limit is defined by the server.

The feed endpoint _must_ add a `next` link, when the `items` array is not empty.

The `next` Link must include an `offset` query parameter with the highest `position` in the current page. 
The feed endpoint must omit the `next` link, when the `items` array is empty.

The feed endpoint _must_ support fetching items starting after a request position using the `offset` query paramter.

The feed endpoint _must_ add a `self` link with the requested offset.



## Model

### Links

Link    | Description
---     | ---
`self`  | The link of the current page. Poll this link periodicly with some delay, when no `next` link is set.
`next`  | The link indicates, that more data is available. Call this link immediatelly when all items have been processed.

### Items

Field    | Type   | Optional  | Description
---      | ---    | ---       | ---
`position` | Number :warning: | Mandatory | The `position` must be strictly monotonic increasing. Values may be skipped. Used to query feed items.
`meta`   | Object | Mandatory | Metadata about the item.
`data`   | Object | Optional  | The payload of the item. May be missing, e.g. when the item represents a tombstone.


### Meta

Field    | Type   | Optional  | Description
---      | ---    | ---       | ---
`type`   | String | Mandatory | The type of the item. A feed may contain different item types, especially if it is an event feed. It is recommended to use a namespace, such as a hierarchical naming pattern.
`id` :warning:    | String | Mandatory | A unique identifier (usually a UUID) for this item. Used for idempotency checks.
`key` :warning:   | String | Optional  | A key that may be used for compaction and deletion. Usually some business object number (such as an order number).
`operation` | String | Optional | `put` indicates that the item for the given _key_ was created or updated. `delete` requires the client to delete all data for this _type_ and _key_. Defaults to `put`, if omitted. 
`created` | String | Mandatory | The timestamp of the item addition to the feed. ISO 8601 UTC date and time format.

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

Records may be deleted, when another record was added to the feed with the same key.

:warning: TODO 


## Authentication

:warning: TODO 


## Filter

Servers _may_ support filtering.

:warning: TODO which query paramter

Entries are still included, but th payload is omitted.
A field `"filtered": true` is added to the record.


## Design Decisions

### Static Pagination vs. start-from Semantics

:warning: TODO 

Static pages are simpler than dynamic pagination with server-side filtering.

- Cachable
- Easier to implement efficiently
- Easier to debug

### Following JSON:API conventions

JSON:API has sensible defaults to reprensentate collection data in JSON.

Exceptions to JSON:API

- The mediatype remains `application/json` (and not `application/vnd.api+json`) for broader library support.
- Payload of record is inline, rather in a separate `included` array.

Alternatives

- HAL was considerd, but to limited and not widly supported.


### Why not RSS/ATOM?

:warning: TODO 

RSS and ATOM are the archetypes for REST Feeds.
Their main concepts are adopted.

Yet, they where created to publish news articles in the web.
Their model with attributes, such as `author`, `title`, or `summary` simply doesn`t fit well for general purpose data feeds.

Plus, they enforce the usage of XML.
Nowadays, it is rather common to use JSON as an exchange data format.
Or event better, to let the client decide witch format to use.

### Top Level object or array

`[]` vs. `items: 

### No client defined limit

- caching
- protection
- simplicity
- storage options

## TODOs

- immutability
- Caching Strategy https://wiki.innoq.com/display/CC/FINT+2.+Feed+Basics#FINT2.FeedBasics-2.5.Cachingstrategy
- Long-Polling
- Idempotency
- itemId / position as String or Number?
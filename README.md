# REST Feeds

Publish data and events using HTTP and REST.

## Why

Information systems (such as microservices or [Self-contained Systems](https://scs-architecture.org/) communicate with other systems to publish their data or events. 
E.g. when a order was made in an online shop, the order must be forwarded to the fulfillment system to ship the order.

Often this integration is made with technologies, like:

- Synchronous API calls (REST-APIs, SOAP Web Services, BAPI, ...)
- Message Brokers (Kafka, JMS, MQ, SQS, ...)
- Database Integration

All these integration strategies lead to tight coupling with technical and organizational dependencies and all its problems. 
Think of ownership, availability, resilience, changeability, and firewalls.

:warning: TODO What makes REST Feeds great.

## Concepts

:warning: TODO add some image

* HTTP
* Polling-based
* Content-Negotiation, with JSON as default
* Static Pages

:warning: TODO explain the concepts

## Example

```
GET /orders/last
Accept: application/json
```

```json
200 OK
Content-Type: application/json
{
  "links": {
    "self": "/orders/34335",
    "prev": "/orders/34334",
    "first": "/orders/1",
    "lookup": "/orders/lookup?id={id}"
  },
  "data": [
    {
      "type": "application/vnd.example.order",
      "id": "948e726e-d38e-4b28-affa-115d024c9d13", // unique id, TBD: sequence???
      "operation": "put", // put (default, if omitted) | delete
      "key": "73264289", // business object id. May be used for compaction.
      "created": "2019-12-16T08:41:59Z", // ISO 8601 UTC timestamp
      "payloadField1" : "xxx"
    }
  ]
}
```

## Content Negotiation

Every consumer and provider _must_ support the media type `application/json`.
It is the default and used, when the `Accept` header is missing or not supported.

Further media types may be used, when supported by client and server:

* `text/html` to render feed in browser
* `application/atom+xml` to support feed readers
* `TODO` to support streaming
* `multipart/mixed` to have a more HTTP native representation
* `TODO` Protobuf or Avro to minimize traffic
* any other

## Pages and Polling

:warning: TODO 


## Lookup

:warning: TODO 


## Data Replication or Events

:warning: TODO 


## Tombstone

:warning: TODO 


## Compaction

Entries may be deleted, when another entry was added to the feed with the same key.

:warning: TODO 


## Authentication

:warning: TODO 


## Filter

Servers _may_ support filtering.

:warning: TODO which query paramter

Entries are still included, but th payload is omitted.
A field `"filtered": true` is added to the entry.


## Design Decisions

### Static Pagination

Static pages are simpler than dynamic pagination with server-side filtering.

- Cachable
- Easier to implement efficiently
- Easier to debug

### Following JSON:API conventions

JSON:API has sensible defaults to reprensentate collection data in JSON.

Exceptions to JSON:API

- The mediatype remains `application/json` (and not `application/vnd.api+json`) for broader library support.
- Payload of entries is embedded in a `data` object, rather in a separate `included` array.

Alternatives

- HAL was considerd, but to limited.


### Why not RSS/ATOM?

:warning: TODO 

RSS and ATOM are the archetypes for REST Feeds.
Their main concepts are adopted.

Yet, they where created to publish news articles in the web.
Their model with attributes, such as `author`, `title`, or `summary` simply doesn`t fit well for general purpose data feeds.

Plus, they enforce the usage of XML.
Nowadays, it is rather common to use JSON as an exchange data format.
Or event better, to let the client decide witch format to use.



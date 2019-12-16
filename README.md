# REST Feeds

Publish data and events in independent systems using HTTP and REST.

## Why

Information systems (such as microservices or [Self-contained Systems](https://scs-architecture.org/) communicate to other systems and publish their data or events. 
E.g. when a order was made in an online shop, the order must be forwarded to the fulfillment system to ship the order.

Often this integration is made with technologies, like:

- Synchronous API calls (REST-APIs, SOAP Web Services, BAPI, ...)
- Message Brokers (Kafka, JMS, MQ, SQS, ...)
- Database Integration

All these integration strategies lead to high coupling with technical and organizational dependencies. Think of ownership, availability, resilience, 



## Concepts

* HTTP
* Polling-based
* Content-Negotiation, with JSON as default
* Static Pages



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
      "data": {

      }
    }
  ]
}
```

## Content Negotiation

Every consumer and provider _must_ support the media type `application/json`.
It is used, if the `Accept` header is missing or not supported.

Further media types may be used, when supported by client and server:

* `text/html` to render feed in browser
* `application/atom+xml` to support feed readers
* `multipart/mixed` to have a more HTTP native representation
* Protobuf
* Avro
* any other


## Filter

Servers _may_ support filtering.

Filtering is used 

Entries are still included, but th payload is omitted.
A field `"filtered": true` is added to the entry.

## Tombstone


## Compaction

Entries may be deleted, when another entry was added to the feed with the same key.




##  ISO 8601




## Design Decisions

### Static Pagination

Static pages are simpler than dynamic pagination.

- Cachable
- Easier to implement efficiently
- Easier to debug

### Following JSON:API conventions

JSON:API has sensible defaults to reprensentate collection data in JSON.

Exceptions to JSON:API

- The mediatype remains `application/json` (and not `application/vnd.api+json`) for broader library support.
- Payload of entries is embedded in a `data` object, rather in a separate `included` array.

Alernatives

- HAL was considerd




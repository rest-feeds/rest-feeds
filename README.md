# REST Feeds

Publish data and events asynchronously using HTTP and REST.

This site describes the concept of REST feeds and proposes a data model.


## Example

A REST feed provides access to resources in a chronological sequence of changes.

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

Repeat the request with the `next` link of the last processed item. 

```http
GET /orders?offset=127
Accept: application/json

200 OK
Content-Type: application/json
[]
```

The server uses [long polling](#feed-endpoint).
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
With long polling, the client gets updates immediatelly without delay.

A REST feed complies to these principles:

* HTTP(S) as transfer protocol
* Polling-based, client initiated GET requests, only
* Paged with link to the next page
* Content-Negotiation, with `application/json` as default

REST feeds enable asynchronously decoulped systems without shared infrastructure.

REST feeds supports both, data feeds and event feeds.

### Data Feeds

_Data feeds_ are used to share domain objects (aggregates) with other systems. 
Downstream systems can replicate the data for reference and lookups.
Usually a data feed contains only entries of one `type`.

Typical examples: 

- _Customers_
- _Products_
- _Stock_

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
The `next` Link must return subsequent items.

The feed endpoint _must_ implement [long polling](https://en.wikipedia.org/wiki/Push_technology#Long_polling).
If the server has no newer items, the server holds the request open until a new item arrive and then immediatelly send the response.
The server _should_ send an empty response, if no new item arrived after N seconds (5 seconds recommended).

## Client Behaviour

The initial stored link is the feed endpoint URL without query parameters.

Pseudocode:

```python
link = "https://feed.example.com/orders"
while true:
  response = GET link
  for item in response:
    process item
    link = item.next
```

Note that there is no sleep step necessary.

The client has the responsibility to persist the `next` link of the last processed item.

The client`s item handling must be idempotent, as the processing of further items or the persistence operation may fail. 


## Model

The response contains an array of _items_.


Field    | Type   | Mandatory | Description
---      | ---    | ---       | ---
`id`     | String | Mandatory | A unique value (such as a UUID) for this item. Can be used to implement deduplication/idempotency handling in downstream systems.
`next`   | String | Mandatory | A link to subsequent items (without the present item).
`type`   | String | Mandatory | The type of the item. Usually used to deserialze the payload. A feed may contain different item types, especially if it is an event feed. It is recommended to use a namespaced [media type](https://en.wikipedia.org/wiki/Media_type).
`uri`    | String | Mandatory | A [URI](https://en.wikipedia.org/wiki/Uniform_Resource_Identifier) to this resource. Doesn't have to be unique within the feed. Doesn`t need to actually exist.
`method` | String | Optional | `PUT` indicates that the resource for the _uri_ was created or updated. `DELETE` indicates that the resource for the _uri_ was deleted. Defaults to `PUT`, if omitted. 
`timestamp` | String | Mandatory | The item addition timestamp. ISO 8601 UTC date and time format.
`data`   | Object | Optional  | The payload of the item. May be missing, e.g. when the method was `DELETE`.

Further meta data may be added, e.g. for traceability.


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



## Deletion

When a domain object was deleted ([GDPR](https://gdpr-info.eu/)), the server _must_ append a `DELETE` item with the same `uri` as domain object to delete.

Clients _must_ delete this domain object or otherwise handle the removal.

```
{
  "id": "8a22af5e-d9ea-4e7f-a907-b5e8687800fd",
  "next": "/orders?offset=321",
  "type": "application/vnd.com.example.order",
  "uri": "/orders/777777",
  "method": "DELETE",
  "timestamp": "2019-12-17T07:07:777Z"
}
```

The server _should_ start a [compaction](#compaction) run afterwards to delete previous items for the same URI.

## Compaction

Items _may_ be deleted from the feed, when another item was added to the feed with the same `uri`.

The server _must_ handle next links, when the requested item has been deleting by returning the next higher items.

## Authentication

Feed endpoints _may_ be protected with [HTTP authentication](https://developer.mozilla.org/en-US/docs/Web/HTTP/Authentication). 

The most common authentication schemes are [Basic](https://tools.ietf.org/html/rfc7617) and [Bearer](https://tools.ietf.org/html/rfc6750).

The server _may_ filter feed items based on the principal.
When filtering is applied, [caching](#caching) may be unfeasible.


## Filter

Feed endpoints _may_ support the `filter` query parameter to return a subset of items, following the recommendations in [JSON:API filtering](https://jsonapi.org/recommendations/#filtering).

Example:

```
GET /orders?offset=123&filter[type]=com.example.order&filter[id]=123456,123457
```

When filtering is applied, [caching](#caching) may be unfeasible.

## Caching

Feed endpoints _may_ set a `Cache-Control: public, max-age=31536000` [header](https://devcenter.heroku.com/articles/increasing-application-performance-with-http-cache-headers), when a page is full and will not be modified any more.


## More information

- [FAQ](FAQ.md)
- [Design decisions](ADRs.md)

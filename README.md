# rest-feeds

Asynchronous data replication and event streaming with plain REST/HTTP.

## Example

A REST feed provides access to resources in a chronological sequence of changes.

```
GET /movies
Accept: application/json

200 OK
Content-Type: application/json
[{
  "id": "11b592ae-490f-4c07-a174-04db33c2df70",
  "next": "/movies?offset=126",
  "type": "application/vnd.org.themoviedb.movie",
  "resource": "/movies/18",
  "timestamp": "2019-12-16T08:41:519Z",
  "data": {
    "original_title":"The Fifth Element",
    "popularity":26.163
  }
},{
  "id": "64e11a7a-0e40-426c-8d81-259d6f6ab74e",
  "next": "/movies?offset=127",
  "type": "application/vnd.org.themoviedb.movie",
  "resource": "/movies/12",
  "timestamp": "2019-12-16T09:12:421Z",
  "data": {
    "original_title":"Finding Nemo",
    "popularity":23.675
  }
}]
```

The number of items returned by the server is limited, e. g. to 100 items per request.
Repeat the request with the `next` link of the last processed item. 
Note that this is an example of a [data feed](#data-feed) where resources can be updated and deleted:

```
GET /movies?offset=127
Accept: application/json

200 OK
Content-Type: application/json
[{
  "id": "756e21c9-4ebd-4354-8f7d-85cd7d2bc4ec",
  "next": "/movies?offset=128",
  "type": "application/vnd.org.themoviedb.movie",
  "resource": "/movies/18",
  "timestamp": "2019-12-17T11:09:122Z",
  "data": {
    "original_title":"The Fifth Element",
    "popularity":27.011
  }
},{
  "id": "e510d24e-bf06-4f6a-b6db-5744f6ff2591",
  "next": "/movies?offset=129",
  "type": "application/vnd.org.themoviedb.movie",
  "resource": "/movies/12",
  "method": "DELETE",
  "timestamp": "2019-12-18T17:00:786Z"
}]
```

The server implements [long polling](#feed-endpoint).
When there are no newer items, the server holds the connection open until new data or a timeout arrives. 

Example for no new data after 5 seconds:

```
GET /movies?offset=129
Accept: application/json

200 OK
Content-Type: application/json
[]
```

The client continues polling this link until new items are received.

## REST Feeds

REST feeds provide access to resources in a chronological sequence of changes using plain HTTP(S).

A REST feed complies to these principles:

* HTTP(S) as transfer protocol
* Clients poll the feed endpoint for new items
* Paged results with links to further items
* Content-Negotiation, with `application/json` as default
* Feed items are immutable.
* Items are always appended.

REST feeds enable asynchronously decoupled systems without shared infrastructure.

REST feeds can be used for data replication (_data feeds_) and event streaming (_event feeds_).

### Data Feeds

_Data feeds_ are used to share resources (master data, domain objects, aggregates) with other systems for data replication.

Every update to the resource leads to a new entry of the full current state in the feed.

A data feed must contain every resource (identified through its `resource`) at least once. 

### Event Feeds

_Event feeds_ are used to publish [domain events](https://www.innoq.com/en/blog/domain-events-versus-event-sourcing/#domainevents) that have happened in a domain.
Event feeds often contain different entry `type`s. 

Events may be deleted once they are outdated or useless.

## Specification

### Feed Endpoint

The feed endpoint _must_ return items in a strictly ascending order of addition to the feed.

The feed endpoint _must_ support fetching items without any query parameters, starting with the first items.

The feed endpoint _may_ choose to limit the number of items returned in a response to a subset of the whole set available. 

The feed endpoint _must_ add a `next` link to every returned item.
The `next` Link must return subsequent items.

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
      link = item.next
  except:
    wait N seconds
```

The client _must_ persist the `next` link of the last processed item.

The client's item handling _must_ be idempotent (_at-least-once_ delivery semantic). 
The `id` _may_ be used for idempotency checks.

The client _must_ implement an exception handler that delays the next request, to protect the server in case of connection or processing errors.

### Model

The response contains an array of _items_.

Field    | Type   | Mandatory | Description
---      | ---    | ---       | ---
`id`     | String | Mandatory | A unique value (such as a UUID) for this item. It can be used to implement deduplication/idempotency handling in downstream systems.
`next`   | String | Mandatory | A link to subsequent items. Fetching the link returns a (paged) collection with subsequent items (without the current item). May be absolute or relative.
`type`   | String | Mandatory | The type of the item. Usually used to deserialize the payload. A feed may contain different item types, especially if it is an event feed. It is recommended to use a namespaced [media type](https://en.wikipedia.org/wiki/Media_type).
`resource` | String | Optional | A [URI](https://en.wikipedia.org/wiki/Uniform_Resource_Identifier) to the resource, the feed item refers to. It doesn't have to be unique within the feed.
`method` | String | Optional | The HTTP equivalent method type that the feed item performs on the `resource`. `PUT` indicates that the _resource_ was created or updated. `DELETE` indicates that the  _resource_ was deleted. Defaults to `PUT`.
`timestamp` | String | Mandatory | The item addition timestamp. ISO 8601 UTC date and time format.
`data`   | Object | Optional  | The payload of the item. May be missing, e.g. when the method was `DELETE`.

Further metadata may be added, e.g. for traceability.


### Content Negotiation

Every consumer and provider _must_ support the media type `application/json`.
It is the default and used when the `Accept` header is missing or not supported.

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
  "id": "8a22af5e-d9ea-4e7f-a907-b5e8687800fd",
  "next": "/movies?offset=321",
  "type": "application/vnd.org.themoviedb.movies",
  "resource": "/movies/777777",
  "method": "DELETE",
  "timestamp": "2019-12-17T07:07:777Z"
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

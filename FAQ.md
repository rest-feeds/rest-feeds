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


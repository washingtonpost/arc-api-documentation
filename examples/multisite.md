# Taking Advantage of Multi-Site Features in ANS 0.6.0

## Introduction

In ANS 0.6.0, Arc Publishing has implemented the first set of features for publishing content across multiple publications (or "websites") within a single umbrella organization.

ANS 0.6.0 will be available for all Arc customers on January 25th.

This document demonstrates how to take advantage of the new multi-site features in Arc Publishing using the public API.

## Goals

This guide will show how to use the Arc APIs to:

* Create distinct section taxonomies and navigation trees for each website.
* Categorize a single document in different sections in different websites.
* Publish a single document to any set of websites within an organization.
* Query the Content API within the context of a single website.
* Assign a different URL for each website to a single document.
* Create different URL formatting rules for each website.

## Prerequisites

* Knowledge of HTTP and cURL (or other HTTP client)
* JSON editor
* An active Arc account and basic auth credentials
* Familiarity with the publishing flow in Arc (See (https://github.com/washingtonpost/arc-api-documentation/blob/master/examples/publishing.md))

## Terminology

* `organization` -- Each Arc customer is considered to be a single organization, and given an *organization id*. Your organization id is embedded in the domain you use to access the Arc API. For example, if your organization id is `washpost`, you would access the APIs at (https://api.washpost.arcpublishing.com).

* `website` -- A website is a distinct publishing destination within an organization. An organization may have up to 100 websites.

* `section`-- A section is a hierarchical node within a website. The usually correspond to different verticals within a publication, for example, "Sports" or "Politics."

* `document` -- A document is any piece of content that a user can publish to a website. An ANS document can be a story, image gallery, or video. Documents are searchable in the Content API and renderable using PageBuilder templates. See [the ANS document schema](https://github.com/washingtonpost/ans-schema). This guide will focus on story documents.


## Accessing the APIs

See ["Accessing the APIs" in the Publishing a Document Example](https://github.com/washingtonpost/arc-api-documentation/blob/master/examples/publishing.md#accessing-the-apis). We'll make the same assumptions here.

Note for several key Arc APIs, new API versions have been released. In order to use multisite features, you'll need to use the following API versions:

* URL API: v2 or higher (released 2018.01)
* Site API: v3 or higher (released 2017.10)
* Content API: v4 or higher (release 2018.01)
* Story API: users can continue using the v2 API

For Content API, Story API and URL API endpoints that expect or return an ANS document, ANS 0.6.0 or higher is also required to use multisite features.

## Create distinct section taxonomies

We'll start by creating the websites and sections for our organization. Websites and sections are efined via the Site API. Full documentation for the Site API v3 is available at https://${ORGANIZATIONID}.arcpublishing.com/siteservice/doc/ (replacing ${ORGANIZATIONID} with your organization id.)

Let's create two websites called "The River City News" and "The Mountain Village Gazette."

```bash
curl  -X PUT https://api.thepost.arcpublishing.com/site/v3/website/rivercitynews -d '{
  "_id":"rivercitynews",
  "display_name": "The River City News"
}'

{"_id":"rivercitynews","display_name":"The River City News"}

curl -X PUT https://api.thepost.arcpublishing.com/site/v3/website/mountainvillagegazette -d '{
  "_id":"mountainvillagegazette",
  "display_name": "The Mountain Village Gazette"
}'

{"_id":"mountainvillagegazette","display_name":"The Mountain Village Gazette"}
```

Now let's add two sections to The River City News: "News" and "Sports."

```bash
curl -X PUT 'https://api.thepost.arcpublishing.com/site/v3/website/rivercitynews/section/?_id=/news' -d '{
  "_id":"/news",
  "_website": "rivercitynews",
  "site": {"site_title": "News" },
  "parent": { "default": "/" }
}'

{"_id":"/news","_website":"rivercitynews","site":{"site_title":"News"},"parent":{"default":"/"},"inactive":false}

curl -X PUT 'https://api.thepost.arcpublishing.com/site/v3/website/rivercitynews/section/?_id=/sports' -d '{
  "_id":"/sports",
  "_website": "rivercitynews",
  "site": {"site_title": "Sports" },
  "parent": { "default": "/" }
}'

{"_id":"/sports","_website":"rivercitynews","site":{"site_title":"Sports"},"parent":{"default":"/"},"inactive":false}
```

We'll add two similar sections to The Mountain Village Gazette. But since Mountain Village residents are extremely passionate about croquet, we'll also add a section for their croquet team, The Mountain Goats.

```bash
curl -X PUT 'https://api.thepost.arcpublishing.com/site/v3/website/mountainvillagegazette/section/?_id=/news' -d '{
  "_id":"/news",
  "_website": "mountainvillagegazette",
  "site": {"site_title": "News" },
  "parent": { "default": "/" }
}'

{"_id":"/news","_website":"mountainvillagegazette","site":{"site_title":"News"},"parent":{"default":"/"},"inactive":false}

curl -X PUT 'https://api.thepost.arcpublishing.com/site/v3/website/mountainvillagegazette/section/?_id=/sports' -d '{
  "_id":"/sports",
  "_website": "mountainvillagegazette",
  "site": {"site_title": "Sports" },
  "parent": { "default": "/" }
}'

{"_id":"/sports","_website":"mountainvillagegazette","site":{"site_title":"Sports"},"parent":{"default":"/"},"inactive":false}

curl -X PUT 'https://api.thepost.arcpublishing.com/site/v3/website/mountainvillagegazette/section/?_id=/sports' -d '{
  "_id":"/sports",
  "_website": "mountainvillagegazette",
  "site": {"site_title": "Sports" },
  "parent": { "default": "/" }
}'

{"_id":"/sports/the-mountain-goats","_website":"mountainvillagegazette","site":{"site_title":"The Mountain Goats"},"parent":{"default":"/sports"},"inactive":false}
```

Note that we've declared "The Mountain Goats" to be a child section of "Sports" by setting the value of `parent.default` to "/sports" in the final PUT request.

The sections in our two websites now look like this:

*The River City News*
  * News
  * Sports

*The Mountain Village Gazette*
  * News
  * Sports
    * The Mountain Goats


## Categorize a single document in sections in different websites

Now that our websites and sections are created, we can create a document and assign it to different sections.

Here's a story document that describes a recent croquet game between The Mountain Goats and The River Turtles. It was written by Brooks Robinson of the The River City News, but will also be published by The Mountain Village Gazette. The content of the document is the same for both websites, but it will appear in different sections in each. The River City News would like to highlight the Turtles' victory at the top of the News section, while the The Mountain Village Gazette relegates the story to the dedicated Mountain Goats section. Both newspapers will include the story in their respective Sports sections.

```json
{
  "_id": "ABCDEFGHIJKLMNOPQRSTUVWXYZ",
  "type": "story",
  "version": "0.6.0",

  "revision": {
    "revision_id": "BCDEFGHIJKLMNOPQRSTUVWXYZA"
  },

  "headlines": {
    "basic": "River Turtles Defeat Mountain Goats in Annual Croquet Match"
  },

  "content_elements": [
    {
      "type": "text",
      "content": "In a surprise upset, The River Turtles of River City have defeated their long-time rivals, The Mountain Goats of Mountain Village, in a tightly-contested match lasting over five hours. The final score was 26-25."
    }
  ],

  "credits": {
    "by": [
      {
        "type": "author",
        "version": "0.6.0",
        "name": "Brooks Robinson"
      }
    ]
  },

  "display_date": "2018-01-18T12:00:00Z",

  "taxonomy": {
    "sections": [
      {
        "type": "reference",
        "referent": {
          "type": "section",
          "id": "/sports",
          "website": "rivercitynews"
        }
      },
      {
        "type": "reference",
        "referent": {
          "type": "section",
          "id": "/news",
          "website": "rivercitynews"
        }
      },
      {
        "type": "reference",
        "referent": {
          "type": "section",
          "id": "/sports",
          "website": "mountainvillagegazette"
        }
      },
      {
        "type": "reference",
        "referent": {
          "type": "section",
          "id": "/sports/the-mountain-goats",
          "website": "mountainvillagegazette"
        }
      }
    ]
  },

  "websites": {
    "rivercitynews": {},
    "mountainvillagegazette": {}
  },

  "canonical_website": "rivercitynews"
}

```

### What's going on here?

* `taxonomy.sections` contains a list of sections *across all websites* that this document belongs to. Here, we've specified each section in the normalized format for entry into Story API.
* `websites` contains a dictionary with a key for each website the document is considered to be part of. For now, the value associated with each key is empty. We'll come back to that later.
* `canonical_website` specifies that The River City News was the originating website for this article. This is important for SEO purposes.

Let's submit this as a draft to Story API.

```bash
curl -X PUT http://api.thepost.arcpublishing.com/story/v2/story/ABCDEFGHIJKLMNOPQRSTUVWXYZ --data @/path/to/river-turtles-defeat-mountain-goats.json

{"_id":"ABCDEFGHIJKLMNOPQRSTUVWXYZ","type":"story","version":"0.6.0","content_elements":[{"_id":"2L565CADAZGXBCCJHRCZPA4L2A","type":"text","content":"In a surprise upset, The River Turtles of River City have defeated their long-time rivals, The Mountain Goats of Mountain Village, in a tightly-contested match lasting over five hours. The final score was 26-25."}],"created_date":"2018-01-18T22:15:10.044Z","revision":{"revision_id":"BCDEFGHIJKLMNOPQRSTUVWXYZA","branch":"default"},"last_updated_date":"2018-01-18T22:15:10.044Z","headlines":{"basic":"River Turtles Defeat Mountain Goats in Annual Croquet Match"},"owner":{"id":"thepost"},"display_date":"2018-01-18T12:00:00Z","credits":{"by":[{"type":"author","version":"0.6.0","name":"Brooks Robinson"}]},"websites":{"rivercitynews":{},"mountainvillagegazette":{}},"taxonomy":{"sections":[{"type":"reference","referent":{"type":"section","id":"/sports","website":"rivercitynews"}},{"type":"reference","referent":{"type":"section","id":"/news","website":"rivercitynews"}},{"type":"reference","referent":{"type":"section","id":"/sports","website":"mountainvillagegazette"}},{"type":"reference","referent":{"type":"section","id":"/sports/the-mountain-goats","website":"mountainvillagegazette"}}]},"additional_properties":{"has_published_copy":false},"canonical_website":"rivercitynews"}
```

We can then fetch a denormalized version of this draft document from the Content API's mutli-site endpoint, from either website:

```bash
curl -X GET 'https://api.thepost.arcpublishing.com/content/v4/stories?_id=ABCDEFGHIJKLMNOPQRSTUVWXYZ&published=false&website=rivercitynews'

curl -X GET 'https://api.thepost.arcpublishing.com/content/v4/stories?_id=ABCDEFGHIJKLMNOPQRSTUVWXYZ&published=false&website=mountainvillagegazette'
```

Publishing this draft works the same as in the single-site workflow: create an edition that points to the revision.

```bash
curl -X PUT https://api.thepost.arcpublishing.com/story/v2/story/ABCDEFGHIJKLMNOPQRSTUVWXYZ/edition/default -d '{ "revision_id": "BCDEFGHIJKLMNOPQRSTUVWXYZA" }'

{"_id":"ABCDEFGHIJKLMNOPQRSTUVWXYZ","type":"story","version":"0.6.0","content_elements":[{"_id":"2L565CADAZGXBCCJHRCZPA4L2A","type":"text","content":"In a surprise upset, The River Turtles of River City have defeated their long-time rivals, The Mountain Goats of Mountain Village, in a tightly-contested match lasting over five hours. The final score was 26-25."}],"created_date":"2018-01-18T22:15:10.044Z","revision":{"revision_id":"BCDEFGHIJKLMNOPQRSTUVWXYZA","branch":"default"},"last_updated_date":"2018-01-18T22:15:10.044Z","headlines":{"basic":"River Turtles Defeat Mountain Goats in Annual Croquet Match"},"display_date":"2018-01-18T22:34:28.023Z","credits":{"by":[{"type":"author","version":"0.6.0","name":"Brooks Robinson"}]},"first_publish_date":"2018-01-18T22:34:28.023Z","websites":{"rivercitynews":{},"mountainvillagegazette":{}},"taxonomy":{"sections":[{"type":"reference","referent":{"type":"section","id":"/sports","website":"rivercitynews"}},{"type":"reference","referent":{"type":"section","id":"/news","website":"rivercitynews"}},{"type":"reference","referent":{"type":"section","id":"/sports","website":"mountainvillagegazette"}},{"type":"reference","referent":{"type":"section","id":"/sports/the-mountain-goats","website":"mountainvillagegazette"}}]},"additional_properties":{"has_published_copy":false},"publish_date":"2018-01-18T22:34:28.023Z","canonical_website":"rivercitynews"}

```

## Query the Content API within a website

Now the document can be fetched from Content API with two different values for the `website` query parameter. But it's the same data in both places. How does this help us build out two different websites?

The key comes in the new website-specific query support. Here's an Elasticsearch query to retrieve all the *published* articles in the News section in The River City News:

```json
{
  "query": {
    "bool": {
      "must": [
        {
          "term": { "revison.published": 1 }
        },
        {
          "nested": {
            "path": "taxonomy.sections",
            "query": {
              "bool": {
                "must": [
                  {
                    "term": {
                      "taxonomy.sections._id": "/news"
                    }
                  },
                  {
                    "term": {
                      "taxonomy.sections._website": "rivercitynews"
                    }
                  }
                ]
              }
            }
          }
        }
      ]
    }
  }
}
```

After minifying and URL-encoding this query, we can run it in the Content API like so:


```bash
curl -X GET 'https://api.thepost.arcpublishing.com/content/v4/search?website=rivercitynews&body=%7B%22query%22%3A%7B%22bool%22%3A%7B%22must%22%3A%5B%7B%22term%22%3A%7B%22revision.published%22%3A1%7D%7D%2C%7B%22nested%22%3A%7B%22path%22%3A%22taxonomy.sections%22%2C%22query%22%3A%7B%22bool%22%3A%7B%22must%22%3A%5B%7B%22term%22%3A%7B%22taxonomy.sections._id%22%3A%22%2Fnews%22%7D%7D%2C%7B%22term%22%3A%7B%22taxonomy.sections._website%22%3A%22rivercitynews%22%7D%7D%5D%7D%7D%7D%7D%5D%7D%7D%7D'

{"type":"results","version":"0.6.0","content_elements":[{"type":"story","version":"0.6.0","content_elements":[{"_id":"2L565CADAZGXBCCJHRCZPA4L2A","type":"text","content":"In a surprise upset, The River Turtles of River City have defeated their long-time rivals, The Mountain Goats of Mountain Village, in a tightly-contested match lasting over five hours. The final score was 26-25."}],"created_date":"2018-01-18T22:15:10.044Z","revision":{"revision_id":"BCDEFGHIJKLMNOPQRSTUVWXYZA","editions":["default"],"branch":"default","published":true},"last_updated_date":"2018-01-18T22:15:10.044Z","headlines":{"basic":"River Turtles Defeat Mountain Goats in Annual Croquet Match"},"owner":{"id":"thepost"},"display_date":"2018-01-18T22:34:28.023Z","credits":{"by":[{"type":"author","version":"0.6.0","name":"Brooks Robinson"}]},"first_publish_date":"2018-01-18T22:34:28.023Z","websites":{"rivercitynews":{},"mountainvillagegazette":{}},"taxonomy":{"sections":[{"_id":"/sports","_website":"rivercitynews","type":"section","version":"0.6.0","name":"Sports","path":"/sports","parent_id":"/","parent":{"default":"/"},"additional_properties":{"original":{"_id":"/sports","_website":"rivercitynews","site":{"site_title":"Sports"},"parent":{"default":"/"},"inactive":false}},"_website_section_id":"rivercitynews./sports"},{"_id":"/news","_website":"rivercitynews","type":"section","version":"0.6.0","name":"News","path":"/news","parent_id":"/","parent":{"default":"/"},"additional_properties":{"original":{"_id":"/news","_website":"rivercitynews","site":{"site_title":"News"},"parent":{"default":"/"},"inactive":false}},"_website_section_id":"rivercitynews./news"},{"_id":"/sports","_website":"mountainvillagegazette","type":"section","version":"0.6.0","name":"Sports","path":"/sports","parent_id":"/","parent":{"default":"/"},"additional_properties":{"original":{"_id":"/sports","_website":"mountainvillagegazette","site":{"site_title":"Sports"},"parent":{"default":"/"},"inactive":false}},"_website_section_id":"mountainvillagegazette./sports"},{"_id":"/sports/the-mountain-goats","_website":"mountainvillagegazette","type":"section","version":"0.6.0","name":"The Mountain Goats","path":"/sports/the-mountain-goats","parent_id":"/sports","parent":{"default":"/sports"},"additional_properties":{"original":{"_id":"/sports/the-mountain-goats","_website":"mountainvillagegazette","site":{"site_title":"The Mountain Goats"},"parent":{"default":"/sports"},"inactive":false}},"_website_section_id":"mountainvillagegazette./sports/the-mountain-goats"}]},"additional_properties":{"has_published_copy":false},"publish_date":"2018-01-19T19:31:08.910Z","canonical_website":"rivercitynews","_website_ids":["rivercitynews","mountainvillagegazette"],"publishing":{"scheduled_operations":{"publish_edition":[],"unpublish_edition":[]}},"_id":"ABCDEFGHIJKLMNOPQRSTUVWXYZ"}],"additional_properties":{"took":2,"timed_out":false},"count":1}
```

As expected, the Content API returned the article.

But, what if we change the query to search for articles in News section from The Mountain Village Gazette? The updated query would be:

```json
{
  "query": {
    "bool": {
      "must": [
        {
          "term": { "revision.published": 1 }
        },
        {
          "nested": {
            "path": "taxonomy.sections",
            "query": {
              "bool": {
                "must": [
                  {
                    "term": {
                      "taxonomy.sections._id": "/news"
                    }
                  },
                  {
                    "term": {
                      "taxonomy.sections._website": "themountainvillagegazette"
                    }
                  }
                ]
              }
            }
          }
        }
      ]
    }
  }
}
```

...and the corresponding Content API call would be...


```bash
curl -X GET 'https://api.thepost.arcpublishing.com/content/v4/search?website=mountainvillagegazette&body=%7B%22query%22%3A%7B%22bool%22%3A%7B%22must%22%3A%5B%7B%22term%22%3A%7B%22revision.published%22%3A1%7D%7D%2C%7B%22nested%22%3A%7B%22path%22%3A%22taxonomy.sections%22%2C%22query%22%3A%7B%22bool%22%3A%7B%22must%22%3A%5B%7B%22term%22%3A%7B%22taxonomy.sections._id%22%3A%22%2Fnews%22%7D%7D%2C%7B%22term%22%3A%7B%22taxonomy.sections._website%22%3A%22mountainvillagegazette%22%7D%7D%5D%7D%7D%7D%7D%5D%7D%7D%7D'

{"type":"results","version":"0.6.0","content_elements":[],"additional_properties":{"took":5,"timed_out":false},"count":0}

```

So the story does *not* appear in queries for the News section in The Mountain Village Gazette.

That means this query syntax can be used to create distinct landing pages for sections with the same name across multiple different websites, all while pulling in the same content!


## Assign a distinct URL for each website to a single document.

All of what we've done so far is nice, but we haven't fully addressed how documents make it out onto the web.  If the same article appears in the News section in River City News and the section The Mountain Goats in The Mountain Village Gazette, shouldn't the urls for each website reflect that? And what is the impact on SEO?

In ANS 0.5.8, the only url you had to populate to make your document appear on the web was the canonical url. In ANS 0.6.0, we've also added the ability to save distinct urls per website (called `website_url`) on a single document. Let's use `website_url` to save urls for The River City News and The Mountain Village Gazette on the document above.

Our new document looks like this:

```json
{
  "_id": "ABCDEFGHIJKLMNOPQRSTUVWXYZ",
  "type": "story",
  "version": "0.6.0",

  "revision": {
    "revision_id": "BCDEFGHIJKLMNOPQRSTUVWXYZA"
  },

  "headlines": {
    "basic": "River Turtles Defeat Mountain Goats in Annual Croquet Match"
  },

  "content_elements": [
    {
      "type": "text",
      "content": "In a surprise upset, The River Turtles of River City have defeated their long-time rivals, The Mountain Goats of Mountain Village, in a tightly-contested match lasting over five hours. The final score was 26-25."
    }
  ],

  "credits": {
    "by": [
      {
        "type": "author",
        "version": "0.6.0",
        "name": "Brooks Robinson"
      }
    ]
  },

  "display_date": "2018-01-18T12:00:00Z",

  "taxonomy": {
    "sections": [
      {
        "type": "reference",
        "referent": {
          "type": "section",
          "id": "/sports",
          "website": "rivercitynews"
        }
      },
      {
        "type": "reference",
        "referent": {
          "type": "section",
          "id": "/news",
          "website": "rivercitynews"
        }
      },
      {
        "type": "reference",
        "referent": {
          "type": "section",
          "id": "/sports",
          "website": "mountainvillagegazette"
        }
      },
      {
        "type": "reference",
        "referent": {
          "type": "section",
          "id": "/sports/the-mountain-goats",
          "website": "mountainvillagegazette"
        }
      }
    ]
  },

  "websites": {
    "rivercitynews": {
      "website_url": "/news/river-turtles-defeat-mountain-goats-croquet-match"
    },
    "mountainvillagegazette": {
      "website_url": "/sports/the-mountain-goats/river-turtles-defeat-mountain-goats-croquet-match"
    }
  },

  "canonical_website": "rivercitynews"
}

```


This is the same document we've seen before, but with an updated websites field.

* The River City News wants to publish the document to `/news/river-turtles-defeat-mountain-goats-croquet-match`.
* The Mountain Village Gazette wants to publish the document to `/news/river-turtles-defeat-mountain-goats-croquet-match`.

To save these urls in URL Service, we can post the document to one of the new URL endpoints:

```bash
curl -X POST 'https://api.thepost.arcpublishing.com/url/v2/url/allwebsiteurls' --data @/path/to/river-turtles-defeat-mountain-goats.json

{"websites":{"mountainvillagegazette":{"url":"/sports/the-mountain-goats/river-turtles-defeat-mountain-goats-croquet-match","format":null},"rivercitynews":{"url":"/news/river-turtles-defeat-mountain-goats-croquet-match","format":null}}}

```

We can then re-publish the story:

```bash
curl -X PUT https://api.thepost.arcpublishing.com/story/v2/story/ABCDEFGHIJKLMNOPQRSTUVWXYZ/edition/default -d '{ "revision_id": "BCDEFGHIJKLMNOPQRSTUVWXYZA" }'

{"_id":"ABCDEFGHIJKLMNOPQRSTUVWXYZ","type":"story","version":"0.6.0","content_elements":[{"_id":"2L565CADAZGXBCCJHRCZPA4L2A","type":"text","content":"In a surprise upset, The River Turtles of River City have defeated their long-time rivals, The Mountain Goats of Mountain Village, in a tightly-contested match lasting over five hours. The final score was 26-25."}],"created_date":"2018-01-18T22:15:10.044Z","revision":{"revision_id":"BCDEFGHIJKLMNOPQRSTUVWXYZA","branch":"default"},"last_updated_date":"2018-01-18T22:15:10.044Z","headlines":{"basic":"River Turtles Defeat Mountain Goats in Annual Croquet Match"},"owner":{"id":"thepost"},"display_date":"2018-01-18T12:00:00Z","credits":{"by":[{"type":"author","version":"0.6.0","name":"Brooks Robinson"}]},"websites":{"rivercitynews":{},"mountainvillagegazette":{}},"taxonomy":{"sections":[{"type":"reference","referent":{"type":"section","id":"/sports","website":"rivercitynews"}},{"type":"reference","referent":{"type":"section","id":"/news","website":"rivercitynews"}},{"type":"reference","referent":{"type":"section","id":"/sports","website":"mountainvillagegazette"}},{"type":"reference","referent":{"type":"section","id":"/sports/the-mountain-goats","website":"mountainvillagegazette"}}]},"additional_properties":{"has_published_copy":false},"canonical_website":"rivercitynews"}

```

Now, we can fetch the document by each site's respective url:

```bash

curl -X GET 'https://api.thepost.arcpublishing.com/content/v4/stories/?website=mountainvillagegazette&website_url=/sports/the-mountain-goats/river-turtles-defeat-mountain-goats-croquet-match'

{"_id":"ABCDEFGHIJKLMNOPQRSTUVWXYZ","type":"story","version":"0.6.0","content_elements":[{"_id":"2L565CADAZGXBCCJHRCZPA4L2A","type":"text","content":"In a surprise upset, The River Turtles of River City have defeated their long-time rivals, The Mountain Goats of Mountain Village, in a tightly-contested match lasting over five hours. The final score was 26-25."}],"created_date":"2018-01-18T22:15:10.044Z","revision":{"revision_id":"BCDEFGHIJKLMNOPQRSTUVWXYZA","editions":["default"],"branch":"default","published":true},"last_updated_date":"2018-01-18T22:15:10.044Z","headlines":{"basic":"River Turtles Defeat Mountain Goats in Annual Croquet Match"},"owner":{"id":"thepost"},"display_date":"2018-01-18T22:34:28.023Z","credits":{"by":[{"type":"author","version":"0.6.0","name":"Brooks Robinson"}]},"first_publish_date":"2018-01-18T22:34:28.023Z","websites":{"rivercitynews":{"website_url":"/news/river-turtles-defeat-mountain-goats-croquet-match"},"mountainvillagegazette":{"website_url":"/sports/the-mountain-goats/river-turtles-defeat-mountain-goats-croquet-match"}},"taxonomy":{"sections":[{"_id":"/sports","_website":"rivercitynews","type":"section","version":"0.6.0","name":"Sports","path":"/sports","parent_id":"/","parent":{"default":"/"},"additional_properties":{"original":{"_id":"/sports","_website":"rivercitynews","site":{"site_title":"Sports"},"parent":{"default":"/"},"inactive":false}},"_website_section_id":"rivercitynews./sports"},{"_id":"/news","_website":"rivercitynews","type":"section","version":"0.6.0","name":"News","path":"/news","parent_id":"/","parent":{"default":"/"},"additional_properties":{"original":{"_id":"/news","_website":"rivercitynews","site":{"site_title":"News"},"parent":{"default":"/"},"inactive":false}},"_website_section_id":"rivercitynews./news"},{"_id":"/sports","_website":"mountainvillagegazette","type":"section","version":"0.6.0","name":"Sports","path":"/sports","parent_id":"/","parent":{"default":"/"},"additional_properties":{"original":{"_id":"/sports","_website":"mountainvillagegazette","site":{"site_title":"Sports"},"parent":{"default":"/"},"inactive":false}},"_website_section_id":"mountainvillagegazette./sports"},{"_id":"/sports/the-mountain-goats","_website":"mountainvillagegazette","type":"section","version":"0.6.0","name":"The Mountain Goats","path":"/sports/the-mountain-goats","parent_id":"/sports","parent":{"default":"/sports"},"additional_properties":{"original":{"_id":"/sports/the-mountain-goats","_website":"mountainvillagegazette","site":{"site_title":"The Mountain Goats"},"parent":{"default":"/sports"},"inactive":false}},"_website_section_id":"mountainvillagegazette./sports/the-mountain-goats"}]},"additional_properties":{"has_published_copy":false},"publish_date":"2018-01-19T21:55:33.666Z","canonical_website":"rivercitynews","canonical_url":"/news/river-turtles-defeat-mountain-goats-croquet-match","publishing":{"scheduled_operations":{"publish_edition":[],"unpublish_edition":[]}},"website":"mountainvillagegazette","website_url":"/sports/the-mountain-goats/river-turtles-defeat-mountain-goats-croquet-match"}
```

Note in particular:

```
"canonical_url":"/news/river-turtles-defeat-mountain-goats-croquet-match"
```

The *canonical_url* field was populated by finding the appropriate *website_url* for the *canonical_website*. This field can now be used to reliably indicate the original source url for a story that exists in multiple places. This is important for SEO purposes!

## Create different URL formatting rules for each website.

The above scenario works if the content author has a particular url in mind for each document on each website. Most of the time, however, authors just want to publish a story and have the URL generated for them.  Furthermore, what if different websites have different ideas about how URLs should look? A simple blog might want all articles to have dates in the url, e.g., /2018/01/18/river-turtles-defeat-mountain-goats.html, while a large newspaper might want everything subdivided into sections, like The Mountain Village Gazette above.

Both scenarios can be handled using *URL formats*, which is Arc's solution for automatically generating URLs, and is now fully multi-site compatible. The complete rules for URL generation are beyond the scope of the this document, but can be found the in URL Service API Documentation, as well as in the URL Serice web app.

For now, let's create a URL-formatting rule for The Mountain Village Gazette which would generate the URL we used above:

```bash
curl -X POST 'https://api.thepost.arcpublishing.com/url/v2/format?website=mountainvillagegazette' -d '{ "criteria": "{ \"type\": \"story\" }", "priority": 10, "format": "%website_section._id|trimForwardSlash()%/%headlines.basic|removeWords()|slugify()%/" }'

{"success":true,"r":{"ok":1,"nModified":0,"n":1,"upserted":[{"index":0,"_id":"1516638491271"}]}}
```

And another rule for The River City News. Let's include the display date in this rule:

```bash
curl -X POST 'https://api.thepost.arcpublishing.com/url/v2/format?website=rivercitynews' -d '{ "criteria": "{ \"type\": \"story\" }", "priority": 10, "format": "%website_section._id|trimForwardSlash()%/%display_date|year()%/%display_date|month()%/%display_date|day()%/%headlines.basic|removeWords()|slugify()%/" }'

{"success":true,"r":{"ok":1,"nModified":0,"n":1,"upserted":[{"index":0,"_id":"1516638491272"}]}}
```


We will also need to delete the old URLs we created manually:

```bash
curl -X DELETE 'https://api.thepost.arcpublishing.com/url/v2/url?website=mountainvillagegazette&url=/sports/the-mountain-goats/river-turtles-defeat-mountain-goats-croquet-match'

{"_id":"/sports/the-mountain-goats/river-turtles-defeat-mountain-goats-croquet-match","content_id":"ABCDEFGHIJKLMNOPQRSTUVWXYZ","created_date":"2018-01-19T21:54:57.288Z","last_updated_date":"2018-01-19T21:54:57.288Z"}

curl -X DELETE 'https://api.thepost.arcpublishing.com/url/v2/url?website=rivercitynews&url=/news/river-turtles-defeat-mountain-goats-croquet-match'

{"_id":"/news/river-turtles-defeat-mountain-goats-croquet-match","content_id":"ABCDEFGHIJKLMNOPQRSTUVWXYZ","created_date":"2018-01-19T21:37:41.385Z","last_updated_date":"2018-01-19T21:37:41.385Z"}
```


Finally, let's remove the user-created urls from the story document we made earlier:

```json

{
  "_id": "ABCDEFGHIJKLMNOPQRSTUVWXYZ",
  "type": "story",
  "version": "0.6.0",

  "revision": {
    "revision_id": "BCDEFGHIJKLMNOPQRSTUVWXYZF",
    "parent_id": "BCDEFGHIJKLMNOPQRSTUVWXYZE"
  },

  "headlines": {
    "basic": "River Turtles Defeat Mountain Goats in Annual Croquet Match"
  },

  "content_elements": [
    {
      "type": "text",
      "content": "In a surprise upset, The River Turtles of River City have defeated their long-time rivals, The Mountain Goats of Mountain Village, in a tightly-contested match lasting over five hours. The final score was 26-25."
    }
  ],

  "credits": {
    "by": [
      {
        "type": "author",
        "version": "0.6.0",
        "name": "Brooks Robinson"
      }
    ]
  },

  "display_date": "2018-01-18T12:00:00Z",

  "taxonomy": {
    "sections": [
      {
        "type": "reference",
        "referent": {
          "type": "section",
          "id": "/sports",
          "website": "rivercitynews"
        }
      },
      {
        "type": "reference",
        "referent": {
          "type": "section",
          "id": "/news",
          "website": "rivercitynews"
        }
      },
      {
        "type": "reference",
        "referent": {
          "type": "section",
          "id": "/sports",
          "website": "mountainvillagegazette"
        }
      },
      {
        "type": "reference",
        "referent": {
          "type": "section",
          "id": "/sports/the-mountain-goats",
          "website": "mountainvillagegazette"
        }
      }
    ]
  },

  "websites": {
    "rivercitynews": {
    },
    "mountainvillagegazette": {
    }
  },

  "canonical_website": "rivercitynews"
}
```

And submit it to the URL Service, as we did earlier:

```bash

curl -X POST `https://api.thepost.arcpublishing.com/url/v2/url/allwebsiteurls' --data @/path/to/river-turtles-defeat-mountain-goats.json

{"websites":{},"_errors":{"mountainvillagegazette":"The ANS object is missing the following item required by the url format:  website_section._id.","rivercitynews":"The ANS object is missing the following item required by the url format:  website_section._id."}}
```

Hmmm, that didn't work. We got an error an error about a missing field: `website_section._id`.

### The Formatting Rules

Let's take a closer look at the formatting rules we established at the beginning of this step.

Both rules have the same *criteria* `{ "type": "story:" }` and *priority* (10). The criteria and priority control when our formatting rule gets triggered. For now, all we need to worry about is that our rules for each website are definitely getting triggered.

They differ, however, in their format. River City News' rule looks like this:

* `"%website_section._id|trimForwardSlash()%/%headlines.basic|removeWords()|slugify()%/`

and Mountain Village Gazette's rule looks like this

* `%website_section._id|trimForwardSlash()%/%display_date|year()%/%display_date|month()%/%display_date|day()%/%headlines.basic|removeWords()|slugify()%/`

The general format for rules is to put *expressions* inside a pair of `%` characters. An expression in a URL formatting rule is usually an ANS *field*, optionally followed by a series of *modifiers*. Each modifier is preceded by a `|` character, and modifiers are processed in sequence from left to right.

Knowing this, and with a bit of deduction (or consulting the reference documentation), we can guess that the URLs generated by each of these rules should look something like this:

* `/news/river-turtles-defeat-mountain-goats-annual-croquet-match`
and
* `/sports/the-mountain-goats/2018/01/18/river-turtles-defeat-mountain-goats-annual-croquet-match`

Now we can start to see where the error message comes from. The beginning of each rule requires the `website_section._id` field, but this doesn't exist on our story!

It turns out that `website_section` **is a special field that does not exist as a top-level ANS field**. However, it is *treated* like one in the URL Service as a convenience.  To specify it, we add it in the appropriate `websites` block. Let's modify our story document and try again:

```json

{
  "_id": "ABCDEFGHIJKLMNOPQRSTUVWXYZ",
  "type": "story",
  "version": "0.6.0",

  "revision": {
    "revision_id": "BCDEFGHIJKLMNOPQRSTUVWXYZF",
    "parent_id": "BCDEFGHIJKLMNOPQRSTUVWXYZA"
  },

  "headlines": {
    "basic": "River Turtles Defeat Mountain Goats in Annual Croquet Match"
  },

  "content_elements": [
    {
      "type": "text",
      "content": "In a surprise upset, The River Turtles of River City have defeated their long-time rivals, The Mountain Goats of Mountain Village, in a tightly-contested match lasting over five hours. The final score was 26-25."
    }
  ],

  "credits": {
    "by": [
      {
        "type": "author",
        "version": "0.6.0",
        "name": "Brooks Robinson"
      }
    ]
  },

  "display_date": "2018-01-18T12:00:00Z",

  "taxonomy": {
    "sections": [
      {
        "type": "reference",
        "referent": {
          "type": "section",
          "id": "/sports",
          "website": "rivercitynews"
        }
      },
      {
        "type": "reference",
        "referent": {
          "type": "section",
          "id": "/news",
          "website": "rivercitynews"
        }
      },
      {
        "type": "reference",
        "referent": {
          "type": "section",
          "id": "/sports",
          "website": "mountainvillagegazette"
        }
      },
      {
        "type": "reference",
        "referent": {
          "type": "section",
          "id": "/sports/the-mountain-goats",
          "website": "mountainvillagegazette"
        }
      }
    ]
  },

  "websites": {
    "rivercitynews": {
      "website_section": {
        "type": "section",
        "version": "0.6.0",
        "_id": "/news",
        "name": "News"
      }
    },
    "mountainvillagegazette": {
      "website_section": {
        "type": "section",
        "version": "0.6.0",
        "_id": "/sports/the-mountain-goats",
        "name": "The Mountain Goats"
      }
    }
  },

  "canonical_website": "rivercitynews"
}
```

Re-submit it to the URL Service:

```bash

curl -X POST `https://api.thepost.arcpublishing.com/url/v2/url/allwebsiteurls' --data @/path/to/river-turtles-defeat-mountain-goats.json

{"websites":{"rivercitynews":{"url":"/news/2018/01/18/river-turtles-defeat-mountain-goats-annual-croquet-match/","format":{"_id":"1516639609928","format":"%website_section._id|trimForwardSlash()%/%display_date|year()%/%display_date|month()%/%display_date|day()%/%headlines.basic|removeWords()|slugify()%/","priority":10,"criteria":{"type":"story"}}},"mountainvillagegazette":{"url":"/sports/the-mountain-goats/river-turtles-defeat-mountain-goats-annual-croquet-match/","format":{"_id":"1516638491271","format":"%website_section._id|trimForwardSlash()%/%headlines.basic|removeWords()|slugify()%/","priority":10,"criteria":{"type":"story"}}}}}
```

The response includes the URLs that were generated for each website as well as the rule that was used to generate them.

For completeness, let's also update the document in Story API to remember the `website_section` fields we added, and re-publish.  (We've changed the `revision` field appropriately for this to succeed.)


```bash

curl -X PUT 'https://api.thepost.arcpublishing.com/story/v2/story/ABCDEFGHIJKLMNOPQRSTUVWXYZ' --data @/path/to/river-turtles-defeat-mountain-goats.json

{"_id":"ABCDEFGHIJKLMNOPQRSTUVWXYZ","type":"story","version":"0.6.0","content_elements":[{"_id":"GX4CJVKAAZBUJH7TPKC3LMUMSA","type":"text","content":"In a surprise upset, The River Turtles of River City have defeated their long-time rivals, The Mountain Goats of Mountain Village, in a tightly-contested match lasting over five hours. The final score was 26-25."}],"created_date":"2018-01-18T22:15:10.044Z","revision":{"revision_id":"BCDEFGHIJKLMNOPQRSTUVWXYZF","parent_id":"BCDEFGHIJKLMNOPQRSTUVWXYZA","branch":"default"},"last_updated_date":"2018-01-22T17:54:23.806Z","headlines":{"basic":"River Turtles Defeat Mountain Goats in Annual Croquet Match"},"owner":{"id":"thepost"},"display_date":"2018-01-18T12:00:00Z","credits":{"by":[{"type":"author","version":"0.6.0","name":"Brooks Robinson"}]},"websites":{"rivercitynews":{"website_section":{"type":"section","version":"0.6.0","_id":"/news","name":"News"}},"mountainvillagegazette":{"website_section":{"type":"section","version":"0.6.0","_id":"/sports/the-mountain-goats","name":"The Mountain Goats"}}},"taxonomy":{"sections":[{"type":"reference","referent":{"type":"section","id":"/sports","website":"rivercitynews"}},{"type":"reference","referent":{"type":"section","id":"/news","website":"rivercitynews"}},{"type":"reference","referent":{"type":"section","id":"/sports","website":"mountainvillagegazette"}},{"type":"reference","referent":{"type":"section","id":"/sports/the-mountain-goats","website":"mountainvillagegazette"}}]},"additional_properties":{"has_published_copy":true},"canonical_website":"rivercitynews"}


curl -X PUT 'https://api.thepost.arcpublishing.com/story/v2/story/ABCDEFGHIJKLMNOPQRSTUVWXYZ/edition/default' -d '{"revision_id": "BCDEFGHIJKLMNOPQRSTUVWXYZF" }'

{"_id":"ABCDEFGHIJKLMNOPQRSTUVWXYZ","type":"story","version":"0.6.0","content_elements":[{"_id":"GX4CJVKAAZBUJH7TPKC3LMUMSA","type":"text","content":"In a surprise upset, The River Turtles of River City have defeated their long-time rivals, The Mountain Goats of Mountain Village, in a tightly-contested match lasting over five hours. The final score was 26-25."}],"created_date":"2018-01-18T22:15:10.044Z","revision":{"revision_id":"BCDEFGHIJKLMNOPQRSTUVWXYZF","parent_id":"BCDEFGHIJKLMNOPQRSTUVWXYZA","branch":"default"},"last_updated_date":"2018-01-22T17:54:23.806Z","headlines":{"basic":"River Turtles Defeat Mountain Goats in Annual Croquet Match"},"display_date":"2018-01-18T22:34:28.023Z","credits":{"by":[{"type":"author","version":"0.6.0","name":"Brooks Robinson"}]},"first_publish_date":"2018-01-18T22:34:28.023Z","websites":{"rivercitynews":{"website_section":{"type":"section","version":"0.6.0","_id":"/news","name":"News"}},"mountainvillagegazette":{"website_section":{"type":"section","version":"0.6.0","_id":"/sports/the-mountain-goats","name":"The Mountain Goats"}}},"taxonomy":{"sections":[{"type":"reference","referent":{"type":"section","id":"/sports","website":"rivercitynews"}},{"type":"reference","referent":{"type":"section","id":"/news","website":"rivercitynews"}},{"type":"reference","referent":{"type":"section","id":"/sports","website":"mountainvillagegazette"}},{"type":"reference","referent":{"type":"section","id":"/sports/the-mountain-goats","website":"mountainvillagegazette"}}]},"additional_properties":{"has_published_copy":true},"publish_date":"2018-01-22T17:56:40.912Z","canonical_website":"rivercitynews"}
```

And now we can fetch from the auto-generated URLs in Content API:

```bash
curl -X GET `https://api.thepost.arcpublishing.com/content/v4/stories/?website=rivercitynews&website_url=/news/2018/01/18/river-turtles-defeat-mountain-goats-annual-croquet-match/'

{"_id":"ABCDEFGHIJKLMNOPQRSTUVWXYZ","type":"story","version":"0.6.0","content_elements":[{"_id":"GX4CJVKAAZBUJH7TPKC3LMUMSA","type":"text","content":"In a surprise upset, The River Turtles of River City have defeated their long-time rivals, The Mountain Goats of Mountain Village, in a tightly-contested match lasting over five hours. The final score was 26-25."}],"created_date":"2018-01-18T22:15:10.044Z","revision":{"revision_id":"BCDEFGHIJKLMNOPQRSTUVWXYZF","parent_id":"BCDEFGHIJKLMNOPQRSTUVWXYZA","editions":["default"],"branch":"default","published":true},"last_updated_date":"2018-01-22T17:54:23.806Z","headlines":{"basic":"River Turtles Defeat Mountain Goats in Annual Croquet Match"},"owner":{"id":"thepost"},"display_date":"2018-01-18T22:34:28.023Z","credits":{"by":[{"type":"author","version":"0.6.0","name":"Brooks Robinson"}]},"first_publish_date":"2018-01-18T22:34:28.023Z","websites":{"rivercitynews":{"website_section":{"type":"section","version":"0.6.0","_id":"/news","name":"News"},"website_url":"/news/2018/01/18/river-turtles-defeat-mountain-goats-annual-croquet-match/"},"mountainvillagegazette":{"website_section":{"type":"section","version":"0.6.0","_id":"/sports/the-mountain-goats","name":"The Mountain Goats"},"website_url":"/sports/the-mountain-goats/river-turtles-defeat-mountain-goats-annual-croquet-match/"}},"taxonomy":{"sections":[{"_id":"/sports","_website":"rivercitynews","type":"section","version":"0.6.0","name":"Sports","path":"/sports","parent_id":"/","parent":{"default":"/"},"additional_properties":{"original":{"_id":"/sports","_website":"rivercitynews","site":{"site_title":"Sports"},"parent":{"default":"/"},"inactive":false}},"_website_section_id":"rivercitynews./sports"},{"_id":"/news","_website":"rivercitynews","type":"section","version":"0.6.0","name":"News","path":"/news","parent_id":"/","parent":{"default":"/"},"additional_properties":{"original":{"_id":"/news","_website":"rivercitynews","site":{"site_title":"News"},"parent":{"default":"/"},"inactive":false}},"_website_section_id":"rivercitynews./news"},{"_id":"/sports","_website":"mountainvillagegazette","type":"section","version":"0.6.0","name":"Sports","path":"/sports","parent_id":"/","parent":{"default":"/"},"additional_properties":{"original":{"_id":"/sports","_website":"mountainvillagegazette","site":{"site_title":"Sports"},"parent":{"default":"/"},"inactive":false}},"_website_section_id":"mountainvillagegazette./sports"},{"_id":"/sports/the-mountain-goats","_website":"mountainvillagegazette","type":"section","version":"0.6.0","name":"The Mountain Goats","path":"/sports/the-mountain-goats","parent_id":"/sports","parent":{"default":"/sports"},"additional_properties":{"original":{"_id":"/sports/the-mountain-goats","_website":"mountainvillagegazette","site":{"site_title":"The Mountain Goats"},"parent":{"default":"/sports"},"inactive":false}},"_website_section_id":"mountainvillagegazette./sports/the-mountain-goats"}]},"additional_properties":{"has_published_copy":true},"publish_date":"2018-01-22T17:56:40.912Z","canonical_website":"rivercitynews","canonical_url":"/news/2018/01/18/river-turtles-defeat-mountain-goats-annual-croquet-match/","publishing":{"scheduled_operations":{"publish_edition":[],"unpublish_edition":[]}},"website":"rivercitynews","website_url":"/news/2018/01/18/river-turtles-defeat-mountain-goats-annual-croquet-match/"}
```

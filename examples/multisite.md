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

TODO



## Assign a distinct URL for each website to a single document.

TODO

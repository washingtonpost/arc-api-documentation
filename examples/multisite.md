# Taking Advantage of Multi-Site Features in ANS 0.6.0

## Introduction

In ANS 0.6.0, Arc Publishing has implemented the first set of features for publishing content across multiple publications (or "websites") within a single umbrella organization.

ANS 0.6.0 will be available for all Arc customers on January 25th.

This document demonstrates how to take advantage of the new multi-site features in Arc Publishing using the public API.

## Goals

* Create distinct section taxonomies and navigation trees for each website.
* Categorize a single document in multiple, different sections in different websites.
* Publish any single document to any set of websites within an organization.
* Assign a distinct URL for each website to a single document.

## Prerequisites

* Knowledge of HTTP and cURL (or other HTTP client)
* JSON editor
* An active Arc account and basic auth credentials
* Familiarity with the publishing flow in Arc (See (https://github.com/washingtonpost/arc-api-documentation/blob/master/examples/publishing.md))

## Terminology

* `organization` -- Each Arc customer is considered to be a single organization, and given an *organization id*. Your organization id is embedded in the domain you use to access the Arc API. For example, if your organization id is `washpost`, you would access the APIs at (https://api.washpost.arcpublishing.com).

* `website` -- A website is a distinct publishing destination within an organization. An organization may have up to 100 websites.

* `section`-- A section is a hierarchical node within a website. The usually correspond to different verticals within a publication, for example, "Sports" or "Politics."

* `hierarchy` -- A hierarchy is a set of relationships between nodes that forms a single tree. A single section can appear in each hierarchy only once, however the same section may appear in different hierarchies with distinct section-section relationships in each. For example, you might create a hierarchy "top-nav" to represent a navigation panel at the top of your website, and "footer" to represent a navigation tree at the bottom.

* `document` -- A document is any piece of content that a user can publish to a website. An ANS document can be a story, image gallery, or video. Documents are searchable in the Content API and renderable using PageBuilder templates. See [the ANS document schema](https://github.com/washingtonpost/ans-schema). This guide will focus on story documents.


## Accessing the APIs

See ["Accessing the APIs" in the Publishing a Document Example](https://github.com/washingtonpost/arc-api-documentation/blob/master/examples/publishing.md#accessing-the-apis). We'll make the same assumptions here.

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

We'll add two similar sections to The Mountain Village Gazette. But since Mountain Village residents are extremely passionate about football, we'll also add a section for their football team, The Mountain Goats.

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

# Example: Publishing a Document in Content API

## Goal

To store custom content in an ANS document in the Content API in a searchable manner.

## Prerequisites

* Knowledge of HTTP and cURL (or other HTTP client)
* JSON editor
* An active Arc account and basic auth credentials

## Accessing the APIs

All API calls to Arc go through an authentication layer on a single domain. This domain is derived from your organization's name. For example, if your organization's name is The Post, the API access domain might look like:

https://api.thepost.arcpublishing.com

In this case "thepost" in the name above is known as your *organization ID*.

In addition, for an API request to succeed, you will need to include your organization's Basic Auth credentials.

Most API calls require the HTTP Header: `Content-Type: application/json`

For example, if your organization ID is "thepost" and you were given the authentication username "2017-12" and the authentication password "password", a cURL request to search the Content API would look like this:

```
curl -H "Content-Type: application/json" --user 2017-12:password -X GET https://api.thepost.arcpublishing.com/content/v3/search?q=*
```

Note that the API access domain is available over HTTPS only.

For brevity, the rest of this document will assume that these headers `-H "Content-Type: application/json" --user 2017-12:password` are present in every cURL example.


## Creating your first document

Documents in the Content API adhere to ANS format. The complete syntactical rules for ANS documents are captured in the [ans-schema](https://github.com/washingtonpost/ans-schema) repository.

For now, let's start by creating a document in the Story API.

The first version of our document looks like this:
```
{
  "type": "story",
  "version": "0.5.8",

  "headlines": {
    "basic": "My First Arc Document"
  },

  "subheadlines": {
    "basic": "Created in Arc"
  },

  "content_elements": [
    {
      "type": "text",
      "content": "This document was created via a call to the Story API."
    },
    {
      "type": "text",
      "content": "My favorite animal is the kangaroo."
    }
  ],
  "display_date": "2017-12-11T14:42:51-05:00"
}
```

Note that this document includes:
 * A `type` and `version` field: These are used to validate the document against a matching type schema in ANS. For now, just know that these need to be present.
 * A headline and subheadline (or "hed" and "dek" if you're a journalist) for the story.
 * A `display_date` which includes the *reader-facing* date that should be displayed with the article.
 * An array of pieces of content in the document, called `content_elements`, with two paragraphs, each represented by an object with `"type": "text"`


Start by saving this document locally to `/some/path/on/your/local/computer/my-first-arc-document.json`

We can save this document in Arc by making a call to the Story API.  Using curl, including the headers specified in the last section, it might look like this:

```
curl -X POST https://api.thepost.arcpublishing.com/story/v2/story --data @/some/path/on/your/local/computer/my-first-arc-document.json
```

Response:
```
{"_id":"TLAWPF3RHJAW5LWWJB2DHQXDT4","type":"story","version":"0.5.8","content_elements":[{"_id":"4N7UEA5L4ZCA5K5VWX72KMDZHQ","type":"text","content":"This document was created via a call to the Story API."},{"_id":"VN7B3JC26NGFJOMBA6ZO5HWWME","type":"text","content":"My favorite animal is the kangaroo."}],"created_date":"2017-12-11T19:52:12.736Z","revision":{"revision_id":"MP3MGQ2ZFBCU7FZSLAVBKJKZGA","branch":"default"},"last_updated_date":"2017-12-11T19:52:12.736Z","headlines":{"basic":"My First Arc Document"},"owner":{"id":"staging"},"display_date":"2017-12-11T14:42:51-05:00","subheadlines":{"basic":"Created in Arc"},"additional_properties":{"has_published_copy":false}}
```


The response is the document you've saved.  Note that a bunch of new fields have been added.  These include:

* `last_updated_date` and `created_date` -- system-generated timestamps
* `owner.id` -- your organization id, derived from your basic auth credentials
* `_id`, `revision._id`, and `revision.branch` -- These are used collectively to uniquely identify your document within Arc Publishing.

A key point to note here is that every update to a *story* creates *story revision*.  The `_id` that is returned by this endpoint is the global id for *all versions of this document across time*.  Each subsequent update to this document will produce a new `revision_id` which represents a new revision in the document history.

(Note: your `_id` and `revision.revision_id` will differ from the examples used here.)

To see this in action, let's update the document. The curl request to update a story is a bit different, mainly because we have to ensure we place the new revision in the correct place in the document's history.


Change your local document to look like this:

```
{
  "type": "story",
  "version": "0.5.8",
  "_id": "TLAWPF3RHJAW5LWWJB2DHQXDT4",

  "headlines": {
    "basic": "My First Arc Document, Updated"
  },

  "subheadlines": {
    "basic": "Created in Arc"
  },

  "content_elements": [
    {
      "type": "text",
      "content": "This document was created via a call to the Story API."
    },
    {
      "type": "text",
      "content": "My favorite animal is the armadillo."
    }
  ],
  "display_date": "2017-12-11T14:42:51-05:00",

  "revision": {
    "parent_id": "MP3MGQ2ZFBCU7FZSLAVBKJKZGA"
  }
}
```


Now update the document with a PUT request to Story API:

```
curl -X PUT https://api.thepost.arcpublishing.com/story/v2/story/TLAWPF3RHJAW5LWWJB2DHQXDT4 --data @/some/path/on/your/local/computer/my-first-arc-document.json

{"_id":"TLAWPF3RHJAW5LWWJB2DHQXDT4","type":"story","version":"0.5.8","content_elements":[{"_id":"Q2LIBHZXX5DFFA32GUEEPRI3LQ","type":"text","content":"This document was created via a call to the Story API."},{"_id":"UFV7V5IM4BBCBC2JP2EE6LMJMY","type":"text","content":"My favorite animal is the armadillo."}],"created_date":"2017-12-11T19:52:12.736Z","revision":{"revision_id":"TCVNCLFU75CLVGLUECE2SBIVMI","parent_id":"MP3MGQ2ZFBCU7FZSLAVBKJKZGA","branch":"default"},"last_updated_date":"2017-12-11T20:02:17.313Z","headlines":{"basic":"My First Arc Document, Updated"},"owner":{"id":"staging"},"display_date":"2017-12-11T14:42:51-05:00","subheadlines":{"basic":"Created in Arc"},"additional_properties":{"has_published_copy":false}}
```

A few things have changed between the first request and the second:

* The POST request changed to a PUT request, to indicate we are updating an existing story rather than creating a new one.
* The `_id` was explicitly declared in the URL and in the JSON's `_id` field.
* The `revision.parent_id` was added, and set to the previous story revision's `revision.revision_id`.
* Last but not least, the document text was changed to replace "kangaroo" with "armadillo."

To update the story a third time, the `revision.parent_id` will be changed again to match this new revision's `revision.revision_id`: "TCVNCLFU75CLVGLUECE2SBIVMI".



Let's confirm that our new revision is saved by fetching the latest version of the story from the Story API:

```
curl -X GET https://api.thepost.arcpublishing.com/story/v2/story/TLAWPF3RHJAW5LWWJB2DHQXDT4

{"_id":"TLAWPF3RHJAW5LWWJB2DHQXDT4","type":"story","version":"0.5.8","content_elements":[{"_id":"Q2LIBHZXX5DFFA32GUEEPRI3LQ","type":"text","content":"This document was created via a call to the Story API."},{"_id":"UFV7V5IM4BBCBC2JP2EE6LMJMY","type":"text","content":"My favorite animal is the armadillo."}],"created_date":"2017-12-11T19:52:12.736Z","revision":{"revision_id":"TCVNCLFU75CLVGLUECE2SBIVMI","parent_id":"MP3MGQ2ZFBCU7FZSLAVBKJKZGA","branch":"default"},"last_updated_date":"2017-12-11T20:02:17.313Z","headlines":{"basic":"My First Arc Document, Updated"},"display_date":"2017-12-11T14:42:51-05:00","subheadlines":{"basic":"Created in Arc"},"additional_properties":{"has_published_copy":false}}
```

Indeed, the current version of the story reads "armadillo" instead of kangaroo. But the kangaroo version is still available, we just need to specify the revision id:

```
curl -X GET https://api.thepost.arcpublishing.com/story/v2/story/TLAWPF3RHJAW5LWWJB2DHQXDT4/revision/MP3MGQ2ZFBCU7FZSLAVBKJKZGA

{"_id":"TLAWPF3RHJAW5LWWJB2DHQXDT4","type":"story","version":"0.5.8","content_elements":[{"_id":"4N7UEA5L4ZCA5K5VWX72KMDZHQ","type":"text","content":"This document was created via a call to the Story API."},{"_id":"VN7B3JC26NGFJOMBA6ZO5HWWME","type":"text","content":"My favorite animal is the kangaroo."}],"created_date":"2017-12-11T19:52:12.736Z","revision":{"revision_id":"MP3MGQ2ZFBCU7FZSLAVBKJKZGA","branch":"default"},"last_updated_date":"2017-12-11T19:52:12.736Z","headlines":{"basic":"My First Arc Document"},"display_date":"2017-12-11T14:42:51-05:00","subheadlines":{"basic":"Created in Arc"},"additional_properties":{"has_published_copy":false}}
```

We'll revisit the Story API in a moment, but if you're impatient, you can see more details of the Story API here:
https://arcpublishing.atlassian.net/wiki/spaces/CA/pages/43188271/Story+API+v2+-+Story+Resource



## Searching for the document in Content API.

The most recently edited version of a story is always available in the Content API. (This is sometimes referred to as the "unpublished" or "nonpublished" copy, even if the story has separately been published.)

To fetch it, we can send a request to the Content API like this:

```
curl -X GET https://api.thepost.arcpublishing.com/content/v3/stories?_id=TLAWPF3RHJAW5LWWJB2DHQXDT4

Content with id=TLAWPF3RHJAW5LWWJB2DHQXDT4, branch=default, published=true, type=story was not found.
```

Wait..it's not found, because we haven't published it yet.  So let's look instead for the nonpublished copy:


```
curl -X GET 'https://api.thepost.arcpublishing.com/content/v3/stories?_id=TLAWPF3RHJAW5LWWJB2DHQXDT4&published=false'

{"_id":"TLAWPF3RHJAW5LWWJB2DHQXDT4","type":"story","version":"0.5.8","content_elements":[{"_id":"Q2LIBHZXX5DFFA32GUEEPRI3LQ","type":"text","content":"This document was created via a call to the Story API."},{"_id":"UFV7V5IM4BBCBC2JP2EE6LMJMY","type":"text","content":"My favorite animal is the armadillo."}],"created_date":"2017-12-11T19:52:12.736Z","revision":{"revision_id":"TCVNCLFU75CLVGLUECE2SBIVMI","parent_id":"MP3MGQ2ZFBCU7FZSLAVBKJKZGA","branch":"default","published":false},"last_updated_date":"2017-12-11T20:02:17.313Z","headlines":{"basic":"My First Arc Document, Updated"},"owner":{"id":"staging"},"display_date":"2017-12-11T14:42:51-05:00","subheadlines":{"basic":"Created in Arc"},"additional_properties":{"has_published_copy":false},"publishing":{"scheduled_operations":{"publish_edition":[],"unpublish_edition":[]}}}
```

There it is, with a bit more added metadata, but only by searching for nonpublished documents. How do we actually publish the document we edited?



## Publishing a document

Story API doesn't just store the revisions as we update, it also controls the publishing state of the document.  Documents saved in Story API can be published by creating a *story edition*. A story __edition__ is a essentially a named pointer to a particular story __revision__ indicating which revision is considered to be the published version of a story.

The document we created earlier can published like so:
```
curl -X PUT 'https://api.thepost.arcpublishing.com/story/v2/story/TLAWPF3RHJAW5LWWJB2DHQXDT4/edition/default' -d '{ "revision_id": "MP3MGQ2ZFBCU7FZSLAVBKJKZGA" }'

{"_id":"TLAWPF3RHJAW5LWWJB2DHQXDT4","type":"story","version":"0.5.8","content_elements":[{"_id":"4N7UEA5L4ZCA5K5VWX72KMDZHQ","type":"text","content":"This document was created via a call to the Story API."},{"_id":"VN7B3JC26NGFJOMBA6ZO5HWWME","type":"text","content":"My favorite animal is the kangaroo."}],"created_date":"2017-12-11T19:52:12.736Z","revision":{"revision_id":"MP3MGQ2ZFBCU7FZSLAVBKJKZGA","branch":"default"},"last_updated_date":"2017-12-11T19:52:12.736Z","headlines":{"basic":"My First Arc Document"},"display_date":"2017-12-11T20:53:20.825Z","subheadlines":{"basic":"Created in Arc"},"first_publish_date":"2017-12-11T20:53:20.825Z","additional_properties":{"has_published_copy":false},"publish_date":"2017-12-11T20:53:20.825Z"}
```

This PUT request creates a new story edition called "default" (derived from the URL) that points to the revision we specified (in this case, the earlier kangaroo revision of the document.) There are also two new fields that only exist on the edition: `publish_date` and `first_publish_date`.

Most importantly, the story is now considered published. We can fetch the published version of the story by retrieving the edition we just created:

```
curl -X GET 'https://api.thepost.arcpublishing.com/story/v2/story/TLAWPF3RHJAW5LWWJB2DHQXDT4/edition/default'

{"_id":"TLAWPF3RHJAW5LWWJB2DHQXDT4","type":"story","version":"0.5.8","content_elements":[{"_id":"4N7UEA5L4ZCA5K5VWX72KMDZHQ","type":"text","content":"This document was created via a call to the Story API."},{"_id":"VN7B3JC26NGFJOMBA6ZO5HWWME","type":"text","content":"My favorite animal is the kangaroo."}],"created_date":"2017-12-11T19:52:12.736Z","revision":{"revision_id":"MP3MGQ2ZFBCU7FZSLAVBKJKZGA","branch":"default"},"last_updated_date":"2017-12-11T19:52:12.736Z","headlines":{"basic":"My First Arc Document"},"display_date":"2017-12-11T20:53:20.825Z","subheadlines":{"basic":"Created in Arc"},"first_publish_date":"2017-12-11T20:53:20.825Z","additional_properties":{"has_published_copy":false},"publish_date":"2017-12-11T20:53:20.825Z"}
```

And we can also find the *published* document in the Content API:

```
curl -X GET 'https://api.thepost.arcpublishing.com/content/v3/stories?_id=TLAWPF3RHJAW5LWWJB2DHQXDT4'

{"_id":"TLAWPF3RHJAW5LWWJB2DHQXDT4","type":"story","version":"0.5.8","content_elements":[{"_id":"4N7UEA5L4ZCA5K5VWX72KMDZHQ","type":"text","content":"This document was created via a call to the Story API."},{"_id":"VN7B3JC26NGFJOMBA6ZO5HWWME","type":"text","content":"My favorite animal is the kangaroo."}],"created_date":"2017-12-11T19:52:12.736Z","revision":{"revision_id":"MP3MGQ2ZFBCU7FZSLAVBKJKZGA","editions":["default"],"branch":"default","published":true},"last_updated_date":"2017-12-11T19:52:12.736Z","headlines":{"basic":"My First Arc Document"},"owner":{"id":"staging"},"display_date":"2017-12-11T20:53:20.825Z","subheadlines":{"basic":"Created in Arc"},"first_publish_date":"2017-12-11T20:53:20.825Z","additional_properties":{"has_published_copy":false},"publish_date":"2017-12-11T20:53:20.825Z","publishing":{"scheduled_operations":{"publish_edition":[],"unpublish_edition":[]}}}
```

The nonpublished document and the published document in Content API are different: the nonpublished document is always *the most recent story revision* but the published document is a *static copy of the published edition*.  Updates that we, or anyone else, make to the document by creating new revisions will not be reflected until the edition named "default" is changed -- i.e., until the document is re-published with those changes. Note also that the history of changes to the edition is distinct from the history of revisions. Publishing a document (or unpublishing via DELETE) does not change the revision history.


More information on the Content API is available here: https://arcpublishing.atlassian.net/wiki/spaces/CA/pages/50928390/Content+API


## Creating an Author

Most documents in the Content API have an associated author. An author is a reader-facing entity that represents the original human producer of the content. (E.g., a writer of a story, or the photographer of an image.)

To create an author, send a POST request to the Author API like the following:

```
curl -X POST https://api.thepost.arcpublishing.com/author/v1 -d '{ "id": "engelg", "name": "Gregory Engel", "bio":"A developer at Arc Publishing" }'


{"ok":true,"reason":"Insert successful."}
```


To verify that your author was created as you expected, you can fetch the same _id that you just POSTed:

```
curl -X GET https://api.thepost.arcpublishing.com/author/v1?_id=engelg

{"_id":"engelg2","name":"Gregory Engel","bio":"A developer at Arc Publishing"}
```


And to see how the author you created will be finally represented within a document in the Content API, add the `ans` query parameter:

```
curl -X GET https://api.thepost.arcpublishing.com/author/v1?_id=engelg&ans=true

{"_id":"engelg2","type":"author","version":"0.5.8","name":"Gregory Engel","bio":"A developer at Arc Publishing","additional_properties":{"original":{"_id":"engelg2","name":"Gregory Engel","bio":"A developer at Arc Publishing"}}}
```

(You can see additional documentation on the Author API: https://arcpublishing.atlassian.net/wiki/spaces/CA/pages/39157865/Author+Service+API )


To be useful, however, that author needs to be included in our document.  Let's update our story document again.

Edit your local copy of the story to be:
```
{
  "type": "story",
  "version": "0.5.8",
  "_id": "TLAWPF3RHJAW5LWWJB2DHQXDT4",

  "headlines": {
    "basic": "My First Arc Document, Updated"
  },

  "subheadlines": {
    "basic": "Created in Arc"
  },

  "credits": {
    "by": [
      {
        "type": "reference",
        "referent": {
          "type": "author",
          "id": "engelg",
          "provider": ""
        }
      }
    ]
  },

  "content_elements": [
    {
      "type": "text",
      "content": "This document was created via a call to the Story API."
    },
    {
      "type": "text",
      "content": "My favorite animal is the armadillo."
    }
  ],
  "display_date": "2017-12-11T14:42:51-05:00",

  "revision": {
    "parent_id": "TCVNCLFU75CLVGLUECE2SBIVMI"
  }
}
```

And update it in the Story API:

```
curl -X PUT https://api.thepost.arcpublishing.com/story/v2/story/TLAWPF3RHJAW5LWWJB2DHQXDT4 --data @/some/path/on/your/local/computer/my-first-arc-document.json

{"_id":"TLAWPF3RHJAW5LWWJB2DHQXDT4","type":"story","version":"0.5.8","content_elements":[{"_id":"JLH3F3OPBNH47GXBCROWBYBA7I","type":"text","content":"This document was created via a call to the Story API."},{"_id":"LE3GAV56NFAD3NWL3GALQ46CBU","type":"text","content":"My favorite animal is the armadillo."}],"created_date":"2017-12-11T19:52:12.736Z","revision":{"revision_id":"XKJTEO7I6VCGHLLNAVL6WASGTE","parent_id":"TCVNCLFU75CLVGLUECE2SBIVMI","branch":"default"},"last_updated_date":"2017-12-11T21:08:01.286Z","headlines":{"basic":"My First Arc Document, Updated"},"owner":{"id":"staging"},"display_date":"2017-12-11T14:42:51-05:00","credits":{"by":[{"type":"reference","referent":{"type":"author","id":"engelg","provider":""}}]},"subheadlines":{"basic":"Created in Arc"},"additional_properties":{"has_published_copy":true}}
```

This isn't that useful.  But let's fetch this update from the Content API:

```
curl -X GET 'https://api.thepost.arcpublishing.com/content/v3/stories?_id=TLAWPF3RHJAW5LWWJB2DHQXDT4&published=false'


{"_id":"TLAWPF3RHJAW5LWWJB2DHQXDT4","type":"story","version":"0.5.8","content_elements":[{"_id":"HJIHRU555NHO5K4C2JBK4NKJ4M","type":"text","content":"This document was created via a call to the Story API."},{"_id":"P5DGVUHT4FFDTLYRIP4MQXU7GI","type":"text","content":"My favorite animal is the armadillo."}],"created_date":"2017-12-11T19:52:12.736Z","revision":{"revision_id":"TK5GTKGELJB6RFOPBAIA2OHR2I","parent_id":"XKJTEO7I6VCGHLLNAVL6WASGTE","branch":"default","published":false},"last_updated_date":"2017-12-11T21:14:24.453Z","headlines":{"basic":"My First Arc Document, Updated"},"owner":{"id":"staging"},"display_date":"2017-12-11T14:42:51-05:00","credits":{"by":[{"_id":"engelg","type":"author","version":"0.5.8","name":"Gregory Engel","description":"A developer at Arc Publishing","additional_properties":{"original":{"_id":"engelg","name":"Gregory Engel","bio":"A developer at Arc Publishing"}}}]},"subheadlines":{"basic":"Created in Arc"},"additional_properties":{"has_published_copy":true},"publishing":{"scheduled_operations":{"publish_edition":[],"unpublish_edition":[]}}}
```

All the data we added to the author "engelg" in the Author API is now present in our document. How did that happen?

### Inflation (or Denormalization)

A major feature of the Content API is this "inflation" we just witnessed with the author of this document. Documents that are indexed in the Content API via Story API are "inflated" with components from other api resources in Arc Publishing. The full inflated document is then searchable in the Content API, and all the data is loaded at once when fetching, which is useful for rendering.

The reference format in ANS looks like this:
```
{
  "type": "reference",
  "referent": {
    "type": "author",
    "id": "engelg",
    "provider": ""
  }
}
```

* `referent.type` indicates which resource type to inflate from
* `referent.id` indicates the `_id` field that should be accessed from that api resource
* `referent.provider` is unused here

The document we created put a reference in the `credits.by` field in the document, indicating that this story is "by" this author. Any relationships are allowed, but the most commonly used are "by" and "photos_by". Note that each of these fields is an ordered list, so mutliple authors are possible. (Sub-authors, like "contributors" or "additional_reporting_by" are also possible, but not indexed or searchable in Content API.)

The full list of inflation types and how they can be included in your story is at the bottom of this document.


## Organizing your document in your website with taxonomy

Authors are one kind of reference, but there are several others. One of the most important is the relationship between the story and the rest of the website it lives in.

Arc assumes that websites are organized into a taxonomy structure, with sections like Sports or Politics, and subsections within those, like The Washington Capitals or National. Let's describe our website's taxonomy using the Site API, and then insert our document into a section.

All section taxonomies must reside within a website, so let's start by creating a website.

```
curl -X PUT 'https://api.thepost.arcpublishing.com/site/v3/website/the-post' -d '
{
  "_id": "main-website",
  "display_name": "The Post",
  "is_default_website": true
}'

{"_id":"main-website","display_name":"The Post","is_default_website":true}
```

Now that we have a website, let's add a Science section and Animals and Plants subsections.

```
curl -X POST 'https://api.thepost.arcpublishing.com/site/v2/' -d '
{
  "_id":"/science",
  "_website": "main-website",
  "name": "Science",
  "parent": { "default": "/" },
  "order": { "default": 10 }}'
}'

{"ok":true,"reason":"Insert successful."}

curl -X POST 'https://api.thepost.arcpublishing.com/site/v2/' -d '
{
  "_id":"/science/animals",
  "_website": "main-website",
  "name": "Animals",
  "parent": { "default": "/science" },
  "order": { "default": 10 }}'
}'

{"ok":true,"reason":"Insert successful."}

curl -X POST 'https://api.thepost.arcpublishing.com/site/v2/' -d '
{
  "_id":"/science/plants",
  "_website": "main-website",
  "name": "Plants",
  "parent": { "default": "/science" },
  "order": { "default": 20 }}'
}'

{"ok":true,"reason":"Insert successful."}

```

Now let's indicate that our story about either kangaroos or armadillos is in the /science and /science/animals sections. Edit your local file to look like this:

```json
{
  "type": "story",
  "version": "0.5.8",
  "_id": "TLAWPF3RHJAW5LWWJB2DHQXDT4",

  "headlines": {
    "basic": "My First Arc Document, Updated"
  },

  "subheadlines": {
    "basic": "Created in Arc"
  },

  "credits": {
    "by": [
      {
        "type": "reference",
        "referent": {
          "type": "author",
          "id": "engelg2",
          "provider": ""
        }
      }
    ]
  },

  "content_elements": [
    {
      "type": "text",
      "content": "This document was created via a call to the Story API."
    },
    {
      "type": "text",
      "content": "My favorite animal is the armadillo."
    }
  ],
  "display_date": "2017-12-11T14:42:51-05:00",

  "taxonomy": {
    "sites": [{
      "type": "reference",
      "referent": {
        "type": "site",
        "id": "/science",
        "provider": ""
      }
    }, {
      "type": "reference",
      "referent": {
        "type": "site",
        "id": "/science/animals",
        "provider": ""
      }
    }]
  },

  "revision": {
    "parent_id": "TK5GTKGELJB6RFOPBAIA2OHR2I"
  }
}
```

For more details about the Site API, see this: https://arcpublishing.atlassian.net/wiki/spaces/CA/pages/37814304/Site+Service+API

We added references to two of the new sections we created in `taxonomy.sites`. Let's update the document in the story API:

```
curl -X PUT https://api.thepost.arcpublishing.com/story/v2/story/TLAWPF3RHJAW5LWWJB2DHQXDT4 --data @/some/path/on/your/local/computer/my-first-arc-document.json

{"_id":"TLAWPF3RHJAW5LWWJB2DHQXDT4","type":"story","version":"0.5.8","content_elements":[{"_id":"QUUVGNKWW5G63OGGVIBEQWBKGM","type":"text","content":"This document was created via a call to the Story API."},{"_id":"V7ELMB7OLJBW5NMSQQRSMILXUQ","type":"text","content":"My favorite animal is the armadillo."}],"created_date":"2017-12-11T19:52:12.736Z","revision":{"revision_id":"BWSCXT3Z7ZE2ZMMP54MYRF45TU","parent_id":"TK5GTKGELJB6RFOPBAIA2OHR2I","branch":"default"},"last_updated_date":"2017-12-12T15:07:59.843Z","headlines":{"basic":"My First Arc Document, Updated"},"owner":{"id":"staging"},"display_date":"2017-12-11T14:42:51-05:00","credits":{"by":[{"type":"reference","referent":{"type":"author","id":"engelg2","provider":""}}]},"subheadlines":{"basic":"Created in Arc"},"taxonomy":{"sites":[{"type":"reference","referent":{"type":"site","id":"/science","provider":""}},{"type":"reference","referent":{"type":"site","id":"/science/animals","provider":""}}]},"additional_properties":{"has_published_copy":true}}
```

That's not too interesting by itself, but the inflated version in Content API is somewhat better:

```
curl -X GET https://api.thepost.arcpublishing.com/content/v3/stories?_id=TLAWPF3RHJAW5LWWJB2DHQXDT4&published=false

{"_id":"TLAWPF3RHJAW5LWWJB2DHQXDT4","type":"story","version":"0.5.8","content_elements":[{"_id":"QUUVGNKWW5G63OGGVIBEQWBKGM","type":"text","content":"This document was created via a call to the Story API."},{"_id":"V7ELMB7OLJBW5NMSQQRSMILXUQ","type":"text","content":"My favorite animal is the armadillo."}],"created_date":"2017-12-11T19:52:12.736Z","revision":{"revision_id":"BWSCXT3Z7ZE2ZMMP54MYRF45TU","parent_id":"TK5GTKGELJB6RFOPBAIA2OHR2I","branch":"default","published":false},"last_updated_date":"2017-12-12T15:07:59.843Z","headlines":{"basic":"My First Arc Document, Updated"},"owner":{"id":"staging"},"display_date":"2017-12-11T14:42:51-05:00","credits":{"by":[{"_id":"engelg2","type":"author","version":"0.5.8","name":"Gregory Engel","description":"A developer at Arc Publishing","additional_properties":{"original":{"_id":"engelg2","name":"Gregory Engel","bio":"A developer at Arc Publishing"}}}]},"subheadlines":{"basic":"Created in Arc"},"taxonomy":{"sites":[{"_id":"/science","type":"site","version":"0.5.8","name":"Science","path":"/science","parent_id":"/","additional_properties":{"original":{"_id":"/science","name":"Science","parent":"/","inactive":false,"order":100059}}},{"_id":"/science/animals","type":"site","version":"0.5.8","name":"Animals","path":"/science/animals","parent_id":"/science","additional_properties":{"original":{"_id":"/science/animals","name":"Animals","parent":"/science","inactive":false,"order":100069}}}]},"additional_properties":{"has_published_copy":true},"publishing":{"scheduled_operations":{"publish_edition":[],"unpublish_edition":[]}}}
```

We can see all the data we added to the Science and Animals sections present in the inflated Content API document.

Perhaps more importantly, we can search on some of the data in those sections, even if it is not present in our original story.  Let's query for "science" stories in Content API:

```
curl -X GET https://api.thepost.arcpublishing.com/content/v3/search/?q=Science

{"type":"results","version":"0.5.3","content_elements":[{"type":"story","version":"0.5.8","content_elements":[{"_id":"QUUVGNKWW5G63OGGVIBEQWBKGM","type":"text","content":"This document was created via a call to the Story API."},{"_id":"V7ELMB7OLJBW5NMSQQRSMILXUQ","type":"text","content":"My favorite animal is the armadillo."}],"created_date":"2017-12-11T19:52:12.736Z","revision":{"revision_id":"BWSCXT3Z7ZE2ZMMP54MYRF45TU","parent_id":"TK5GTKGELJB6RFOPBAIA2OHR2I","branch":"default","published":false},"last_updated_date":"2017-12-12T15:07:59.843Z","headlines":{"basic":"My First Arc Document, Updated"},"owner":{"id":"staging"},"display_date":"2017-12-11T14:42:51-05:00","credits":{"by":[{"_id":"engelg2","type":"author","version":"0.5.8","name":"Gregory Engel","description":"A developer at Arc Publishing","additional_properties":{"original":{"_id":"engelg2","name":"Gregory Engel","bio":"A developer at Arc Publishing"}}}]},"subheadlines":{"basic":"Created in Arc"},"taxonomy":{"sites":[{"_id":"/science","type":"site","version":"0.5.8","name":"Science","path":"/science","parent_id":"/","additional_properties":{"original":{"_id":"/science","name":"Science","parent":"/","inactive":false,"order":100059}}},{"_id":"/science/animals","type":"site","version":"0.5.8","name":"Animals","path":"/science/animals","parent_id":"/science","additional_properties":{"original":{"_id":"/science/animals","name":"Animals","parent":"/science","inactive":false,"order":100069}}}]},"additional_properties":{"has_published_copy":true},"publishing":{"scheduled_operations":{"publish_edition":[],"unpublish_edition":[]}},"_id":"TLAWPF3RHJAW5LWWJB2DHQXDT4"}],"additional_properties":{"took":4,"timed_out":false},"count":1,"next":0}
```

Our query for "Science" matched our story document, because "Science" is the name of a section our story is in.


## Adding an image

Most articles aren't just text, they also include an image, sometimes more than one! To include an image in our document, we must first upload it to the Image API, also known as Anglerfish.

First, let's create a separate ANS document to describe our image.

Create a file called `/some/path/on/your/local/computer/my-arc-image.json` on your local machine, and edit it to look like this:

```json
{
  "type": "image",
  "version": "0.5.8",

  "additional_properties": {
    "originalUrl": "https://www.washingtonpost.com/rf/image_480x320/2010-2019/WashingtonPost/2017/12/07/RealEstate/Images/ORI02.JPG"
  }
}
```


(Note that the the Anglerfish API format is a little different, so make sure you do NOT include "Content-Type: application/json" in the HTTP headers)

```
 curl --user 2016-07:93e47ebf87ab9fd07d28af6da82d836bf4a224130fcc0d1df86f7a09d9bbe3580c664fe9fb353af5a36b4fd9b33bd5ceb610ee6688a0f415b92ec0f03c1887fdefe0 -X POST -F "ans=@/Users/engelg/photo.json; type=application/json" 'https://api.staging.arcpublishing.com/photo/api/v2/photos'

 {"_id":"JXJTKHKE6NHH7ODZ5VYEIKQVQM","additional_properties":{"mime_type":"image/jpeg","originalUrl":"https://www.washingtonpost.com/rf/image_480x320/2010-2019/WashingtonPost/2017/12/07/RealEstate/Images/ORI02.JPG","proxyUrl":"/photo/resize/F2zV-JyTwVOqeRD6oUEAJQlaD-4=/arc-anglerfish-staging-staging/JXJTKHKE6NHH7ODZ5VYEIKQVQM.JPG","resizeUrl":"http://anglerfish-staging-thumbor.internal.arc2.nile.works/F2zV-JyTwVOqeRD6oUEAJQlaD-4=/arc-anglerfish-staging-staging/JXJTKHKE6NHH7ODZ5VYEIKQVQM.JPG"},"created_date":"2017-12-12T16:44:07+00:00","height":320,"last_updated_date":"2017-12-12T16:44:07+00:00","licensable":false,"type":"image","version":"0.5.8","width":480}
 ```

This did a couple of things for us:
* The image we specified as `originalUrl` in the source ANS was downloaded and stored in the Image API, accessible in the `url` field
* The height and image were calculated for us in `height` and `width` fields
* Various metadata fields, including `created_date`, `last_updated_date` were added to the document, similar to our story document in Story API

For more detailed information on Anglerfish and the Image API, see the documentation here: https://arcpublishing.atlassian.net/wiki/spaces/ANG/pages/13338195/Anglerfish+API

Now we can include the image in our document. The reference format is the same we used for authors and sections. This time, though, we'll insert the image directly in the content body itself.

Edit the local story json to look like:
```json
{
  "type": "story",
  "version": "0.5.8",
  "_id": "TLAWPF3RHJAW5LWWJB2DHQXDT4",

  "headlines": {
    "basic": "My First Arc Document, Updated"
  },

  "subheadlines": {
    "basic": "Created in Arc"
  },

  "credits": {
    "by": [
      {
        "type": "reference",
        "referent": {
          "type": "author",
          "id": "engelg2",
          "provider": ""
        }
      }
    ]
  },

  "content_elements": [
    {
      "type": "text",
      "content": "This document was created via a call to the Story API."
    },
    {
      "type": "reference",
      "referent": {
        "type": "image",
        "id": "JXJTKHKE6NHH7ODZ5VYEIKQVQM",
        "provider": ""
      }
    },
    {
      "type": "text",
      "content": "My favorite animal is the armadillo."
    }
  ],
  "display_date": "2017-12-11T14:42:51-05:00",

  "taxonomy": {
    "sites": [{
      "type": "reference",
      "referent": {
        "type": "site",
        "id": "/science",
        "provider": ""
      }
    }, {
      "type": "reference",
      "referent": {
        "type": "site",
        "id": "/science/animals",
        "provider": ""
      }
    }]
  },

  "revision": {
    "parent_id": "BWSCXT3Z7ZE2ZMMP54MYRF45TU"
  }
}

```

And PUT the update to Story API:

```
curl -X PUT http://api.thepost.arcpublishing.com/story/v2/story/TLAWPF3RHJAW5LWWJB2DHQXDT4 --data @/some/path/on/your/local/computer/my-first-arc-document.json

{"_id":"TLAWPF3RHJAW5LWWJB2DHQXDT4","type":"story","version":"0.5.8","content_elements":[{"_id":"QOMQZYXMXVDEDIHROL2D3EUSHA","type":"text","content":"This document was created via a call to the Story API."},{"_id":"IA6RGMNIIVF6FCOFTQT22FFRJ4","type":"reference","referent":{"type":"image","id":"JXJTKHKE6NHH7ODZ5VYEIKQVQM","provider":""}},{"_id":"PP7ESJELPRCT5A2YJK74E67XDI","type":"text","content":"My favorite animal is the armadillo."}],"created_date":"2017-12-11T19:52:12.736Z","revision":{"revision_id":"AQXPF7XEGFCQJOZLM3JPO3UO7A","parent_id":"BWSCXT3Z7ZE2ZMMP54MYRF45TU","branch":"default"},"last_updated_date":"2017-12-12T17:07:01.334Z","headlines":{"basic":"My First Arc Document, Updated"},"owner":{"id":"staging"},"display_date":"2017-12-11T14:42:51-05:00","credits":{"by":[{"type":"reference","referent":{"type":"author","id":"engelg2","provider":""}}]},"subheadlines":{"basic":"Created in Arc"},"taxonomy":{"sites":[{"type":"reference","referent":{"type":"site","id":"/science","provider":""}},{"type":"reference","referent":{"type":"site","id":"/science/animals","provider":""}}]},"additional_properties":{"has_published_copy":true}}
```


Finally, let's fetch the denormalized story in Content API to make sure the image data is available:

```
curl -X GET https://api.thepost.arcpublishing.com/content/v3/stories?_id=TLAWPF3RHJAW5LWWJB2DHQXDT4&published=false'

{"_id":"TLAWPF3RHJAW5LWWJB2DHQXDT4","type":"story","version":"0.5.8","content_elements":[{"_id":"QOMQZYXMXVDEDIHROL2D3EUSHA","type":"text","content":"This document was created via a call to the Story API."},{"_id":"JXJTKHKE6NHH7ODZ5VYEIKQVQM","additional_properties":{"galleries":[],"mime_type":"image/jpeg","originalUrl":"https://arc-anglerfish-staging-staging.s3.amazonaws.com/JXJTKHKE6NHH7ODZ5VYEIKQVQM.JPG","proxyUrl":"/photo/resize/F2zV-JyTwVOqeRD6oUEAJQlaD-4=/arc-anglerfish-staging-staging/JXJTKHKE6NHH7ODZ5VYEIKQVQM.JPG","published":false,"resizeUrl":"http://anglerfish-staging-thumbor.internal.arc2.nile.works/F2zV-JyTwVOqeRD6oUEAJQlaD-4=/arc-anglerfish-staging-staging/JXJTKHKE6NHH7ODZ5VYEIKQVQM.JPG","version":0},"created_date":"2017-12-12T16:44:07+00:00","height":320,"last_updated_date":"2017-12-12T16:44:07+00:00","licensable":false,"owner":{"id":"staging","name":"Organization Name Override Goes Here"},"type":"image","url":"https://arc-anglerfish-staging-staging.s3.amazonaws.com/JXJTKHKE6NHH7ODZ5VYEIKQVQM.JPG","version":"0.5.8","width":480},{"_id":"PP7ESJELPRCT5A2YJK74E67XDI","type":"text","content":"My favorite animal is the armadillo."}],"created_date":"2017-12-11T19:52:12.736Z","revision":{"revision_id":"AQXPF7XEGFCQJOZLM3JPO3UO7A","parent_id":"BWSCXT3Z7ZE2ZMMP54MYRF45TU","branch":"default","published":false},"last_updated_date":"2017-12-12T17:07:01.334Z","headlines":{"basic":"My First Arc Document, Updated"},"owner":{"id":"staging"},"display_date":"2017-12-11T14:42:51-05:00","credits":{"by":[{"_id":"engelg2","type":"author","version":"0.5.8","name":"Gregory Engel","description":"A developer at Arc Publishing","additional_properties":{"original":{"_id":"engelg2","name":"Gregory Engel","bio":"A developer at Arc Publishing"}}}]},"subheadlines":{"basic":"Created in Arc"},"taxonomy":{"sites":[{"_id":"/science","type":"site","version":"0.5.8","name":"Science","path":"/science","parent_id":"/","additional_properties":{"original":{"_id":"/science","name":"Science","parent":"/","inactive":false,"order":100059}}},{"_id":"/science/animals","type":"site","version":"0.5.8","name":"Animals","path":"/science/animals","parent_id":"/science","additional_properties":{"original":{"_id":"/science/animals","name":"Animals","parent":"/science","inactive":false,"order":100069}}}]},"additional_properties":{"has_published_copy":true},"publishing":{"scheduled_operations":{"publish_edition":[],"unpublish_edition":[]}}}
```

That's a little hard to read, but cleaned up it looks like:


```json
{
  "_id": "TLAWPF3RHJAW5LWWJB2DHQXDT4",
  "type": "story",
  "version": "0.5.8",
  "headlines": {
    "basic": "My First Arc Document, Updated"
  },
  "subheadlines": {
    "basic": "Created in Arc"
  },
  "credits": {
    "by": [
      {
        "_id": "engelg2",
        "type": "author",
        "version": "0.5.8",
        "name": "Gregory Engel",
        "description": "A developer at Arc Publishing",
        "additional_properties": {
          "original": {
            "_id": "engelg2",
            "name": "Gregory Engel",
            "bio": "A developer at Arc Publishing"
          }
        }
      }
    ]
  },
  "display_date": "2017-12-11T14:42:51-05:00",
  "content_elements": [
    {
      "_id": "QOMQZYXMXVDEDIHROL2D3EUSHA",
      "type": "text",
      "content": "This document was created via a call to the Story API."
    },
    {
      "_id": "JXJTKHKE6NHH7ODZ5VYEIKQVQM",
      "additional_properties": {
        "galleries": [],
        "mime_type": "image/jpeg",
        "originalUrl": "https://arc-anglerfish-staging-staging.s3.amazonaws.com/JXJTKHKE6NHH7ODZ5VYEIKQVQM.JPG",
        "proxyUrl": "/photo/resize/F2zV-JyTwVOqeRD6oUEAJQlaD-4=/arc-anglerfish-staging-staging/JXJTKHKE6NHH7ODZ5VYEIKQVQM.JPG",
        "published": false,
        "resizeUrl": "http://anglerfish-staging-thumbor.internal.arc2.nile.works/F2zV-JyTwVOqeRD6oUEAJQlaD-4=/arc-anglerfish-staging-staging/JXJTKHKE6NHH7ODZ5VYEIKQVQM.JPG",
        "version": 0
      },
      "created_date": "2017-12-12T16:44:07+00:00",
      "height": 320,
      "last_updated_date": "2017-12-12T16:44:07+00:00",
      "licensable": false,
      "owner": {
        "id": "staging",
        "name": "Organization Name Override Goes Here"
      },
      "type": "image",
      "url": "https://arc-anglerfish-staging-staging.s3.amazonaws.com/JXJTKHKE6NHH7ODZ5VYEIKQVQM.JPG",
      "version": "0.5.8",
      "width": 480
    },
    {
      "_id": "PP7ESJELPRCT5A2YJK74E67XDI",
      "type": "text",
      "content": "My favorite animal is the armadillo."
    }
  ],
  "created_date": "2017-12-11T19:52:12.736Z",
  "revision": {
    "revision_id": "AQXPF7XEGFCQJOZLM3JPO3UO7A",
    "parent_id": "BWSCXT3Z7ZE2ZMMP54MYRF45TU",
    "branch": "default",
    "published": false
  },
  "last_updated_date": "2017-12-12T17:07:01.334Z",
  "owner": {
    "id": "staging"
  },
  "taxonomy": {
    "sites": [
      {
        "_id": "/science",
        "type": "site",
        "version": "0.5.8",
        "name": "Science",
        "path": "/science",
        "parent_id": "/",
        "additional_properties": {
          "original": {
            "_id": "/science",
            "name": "Science",
            "parent": "/",
            "inactive": false,
            "order": 100059
          }
        }
      },
      {
        "_id": "/science/animals",
        "type": "site",
        "version": "0.5.8",
        "name": "Animals",
        "path": "/science/animals",
        "parent_id": "/science",
        "additional_properties": {
          "original": {
            "_id": "/science/animals",
            "name": "Animals",
            "parent": "/science",
            "inactive": false,
            "order": 100069
          }
        }
      }
    ]
  },
  "additional_properties": {
    "has_published_copy": true
  },
  "publishing": {
    "scheduled_operations": {
      "publish_edition": [],
      "unpublish_edition": []
    }
  }
}
```




We now have an inflated document:
* With different content in the unpublished and published versions
* With a paragraph, an image, and another paragraph in the body
* That is associated with two sections (/science and /science/animals)
* That has an author

We're almost ready to publish the final version! We just need one more thing: a url for the web.

## Adding a URL

The simplest way to set the url for a document is to POST one to the URL Service. Let's add one to our document:

```
curl -X https://api.thepost.arcpublishing.com/url/v1/url' -d '{"_id":"TLAWPF3RHJAW5LWWJB2DHQXDT4", "canonical_url":"/my-first-arc-document" }'

{"url":"/my-first-arc-document","format":null}
```

This links the relative url "/my-first-arc-document" to the story ID "TLAWPF3RHJAW5LWWJB2DHQXDT4". This url will be added to the document on subsequent updates and publishes.


Let's publish the final revision of our document now to see the `canonical_url` appear.

```
curl -X PUT https://api.thepost.arcpublishing.com/story/v2/story/TLAWPF3RHJAW5LWWJB2DHQXDT4/edition/default -d '{"revision_id":"AQXPF7XEGFCQJOZLM3JPO3UO7A"}'

{"_id":"TLAWPF3RHJAW5LWWJB2DHQXDT4","type":"story","version":"0.5.8","content_elements":[{"_id":"QOMQZYXMXVDEDIHROL2D3EUSHA","type":"text","content":"This document was created via a call to the Story API."},{"_id":"IA6RGMNIIVF6FCOFTQT22FFRJ4","type":"reference","referent":{"type":"image","id":"JXJTKHKE6NHH7ODZ5VYEIKQVQM","provider":""}},{"_id":"PP7ESJELPRCT5A2YJK74E67XDI","type":"text","content":"My favorite animal is the armadillo."}],"created_date":"2017-12-11T19:52:12.736Z","revision":{"revision_id":"AQXPF7XEGFCQJOZLM3JPO3UO7A","parent_id":"BWSCXT3Z7ZE2ZMMP54MYRF45TU","branch":"default"},"last_updated_date":"2017-12-12T17:07:01.334Z","headlines":{"basic":"My First Arc Document, Updated"},"display_date":"2017-12-11T20:53:20.825Z","credits":{"by":[{"type":"reference","referent":{"type":"author","id":"engelg2","provider":""}}]},"subheadlines":{"basic":"Created in Arc"},"first_publish_date":"2017-12-11T20:53:20.825Z","taxonomy":{"sites":[{"type":"reference","referent":{"type":"site","id":"/science","provider":""}},{"type":"reference","referent":{"type":"site","id":"/science/animals","provider":""}}]},"additional_properties":{"has_published_copy":true},"publish_date":"2017-12-12T17:48:43.578Z"}
```

And finally, we can see the resulting document in Content API complete:

```
curl -X GET 'https://api.thepost.arcpublishing.com/content/v3/stories?_id=TLAWPF3RHJAW5LWWJB2DHQXDT4&published=true'

{"_id":"TLAWPF3RHJAW5LWWJB2DHQXDT4","type":"story","version":"0.5.8","content_elements":[{"_id":"QOMQZYXMXVDEDIHROL2D3EUSHA","type":"text","content":"This document was created via a call to the Story API."},{"_id":"IA6RGMNIIVF6FCOFTQT22FFRJ4","type":"reference","referent":{"type":"image","id":"JXJTKHKE6NHH7ODZ5VYEIKQVQM","provider":""}},{"_id":"PP7ESJELPRCT5A2YJK74E67XDI","type":"text","content":"My favorite animal is the armadillo."}],"created_date":"2017-12-11T19:52:12.736Z","revision":{"revision_id":"AQXPF7XEGFCQJOZLM3JPO3UO7A","parent_id":"BWSCXT3Z7ZE2ZMMP54MYRF45TU","editions":["default"],"branch":"default","published":true},"last_updated_date":"2017-12-12T17:07:01.334Z","headlines":{"basic":"My First Arc Document, Updated"},"owner":{"id":"staging"},"display_date":"2017-12-11T20:53:20.825Z","credits":{"by":[{"_id":"engelg2","type":"author","version":"0.5.8","name":"Gregory Engel","description":"A developer at Arc Publishing","additional_properties":{"original":{"_id":"engelg2","name":"Gregory Engel","bio":"A developer at Arc Publishing"}}}]},"subheadlines":{"basic":"Created in Arc"},"first_publish_date":"2017-12-11T20:53:20.825Z","taxonomy":{"sites":[{"_id":"/science","type":"site","version":"0.5.8","name":"Science","path":"/science","parent_id":"/","additional_properties":{"original":{"_id":"/science","name":"Science","parent":"/","inactive":false,"order":100059}}},{"_id":"/science/animals","type":"site","version":"0.5.8","name":"Animals","path":"/science/animals","parent_id":"/science","additional_properties":{"original":{"_id":"/science/animals","name":"Animals","parent":"/science","inactive":false,"order":100069}}}]},"additional_properties":{"has_published_copy":true},"publish_date":"2017-12-12T17:48:43.578Z","canonical_url":"/my-first-arc-document","publishing":{"scheduled_operations":{"publish_edition":[],"unpublish_edition":[]}}}
```

As you can see, our URL is now present in the published document.


## More About URLs

Manually generating a new URL for each story can become both cumbersome and error-prone. As an alternative to the previous step, there is a way to have the URL API generate URLs for us based on the contents of an ANS document.

1. Delete the previously generated URL

```
curl -X DELETE 'https://api.thepost.arcpublishing.com/url/v1/url?url=?url=/my-first-arc-document'

{"_id":"/my-first-arc-document","content_id":"TLAWPF3RHJAW5LWWJB2DHQXDT4","created_date":"2017-12-12T17:48:33.556Z","last_updated_date":"2017-12-12T17:48:33.556Z"}
```

2. Update the story in the Story API

```
curl -X PUT 'https://api.thepost.arcpublishing.com/story/v2/story/TLAWPF3RHJAW5LWWJB2DHQXDT4' --data @/some/path/on/your/local/computer/my-first-arc-document.json
```

3. Fetch and save the denormalized story to your machine with the following:

```
curl -X GET 'https://api.thepost.arcpublishing.com/content/v3/stories?_id=TLAWPF3RHJAW5LWWJB2DHQXDT4&published=false' > /some/path/on/your/local/computer/denormalized.json
```


4. Edit and save the local denormalized.json to include this field:

```
  "publish_date": "2017-12-11T14:42:51-05:00"
```

5. Post the denormalized ANS body to the URL Service:

```
curl -X POST 'https://api.thepost.arcpublishing.com/url/v1/url --data@/some/path/on/your/local/computer/denormalized.json

{"url":"/stories/science/2017/12/11/my-first-arc-document-updated","format":{"_id":"1499701818967","format":"stories%taxonomy.sites[0]._id%/%publish_date|year()%/%publish_date|month()%/%publish_date|day()%/%headlines.basic|slugify()%","priority":20,"criteria":{"type":"story"}}}
```

### What just happened?

* `DELETE /url/v1/url` removed the link between the url and our document in Url Service we had previously created
* `PUT /story/v2/story` updated the document in Story API and Content API, to ensure the old url was gone
* `GET /content/v3/stories` downloaded the denormalized document (now without the `canonical_url` field)
* `publish_date` was then stubbed on the ANS document as a temporary mock
* `POST /url/v1/url` asked URL API to generate a `canonical_url` field from the other data in the document, in this case the `taxonomy.sites` and `publish_date` fields

What fields does the URL API use to generate urls? Fortunately, the answer to that question is configurable. The details of the configuration rules in URL API are beyond the scope of this document. However, you can read about them, as well as the rest of the URL API at: https://arcpublishing.atlassian.net/wiki/spaces/CA/pages/13338275/Url+Service



## Publishing revisited

You may have noticed something odd during the previous publish step.  In the nonpublished edition of the story, the body of the story (represented in ANS as `content_elements`) looks like this:


`https://api.thepost.arcpublishing.com/content/v3/stories?_id=TLAWPF3RHJAW5LWWJB2DHQXDT4&published=false`
```json
  "content_elements": [
    {
      "_id": "QOMQZYXMXVDEDIHROL2D3EUSHA",
      "type": "text",
      "content": "This document was created via a call to the Story API."
    },
    {
      "_id": "JXJTKHKE6NHH7ODZ5VYEIKQVQM",
      "additional_properties": {
        "galleries": [],
        "mime_type": "image/jpeg",
        "originalUrl": "https://arc-anglerfish-staging-staging.s3.amazonaws.com/JXJTKHKE6NHH7ODZ5VYEIKQVQM.JPG",
        "proxyUrl": "/photo/resize/F2zV-JyTwVOqeRD6oUEAJQlaD-4=/arc-anglerfish-staging-staging/JXJTKHKE6NHH7ODZ5VYEIKQVQM.JPG",
        "published": false,
        "resizeUrl": "http://anglerfish-staging-thumbor.internal.arc2.nile.works/F2zV-JyTwVOqeRD6oUEAJQlaD-4=/arc-anglerfish-staging-staging/JXJTKHKE6NHH7ODZ5VYEIKQVQM.JPG",
        "version": 0
      },
      "created_date": "2017-12-12T16:44:07+00:00",
      "height": 320,
      "last_updated_date": "2017-12-12T16:44:07+00:00",
      "licensable": false,
      "owner": {
        "id": "staging",
        "name": "Organization Name Override Goes Here"
      },
      "type": "image",
      "url": "https://arc-anglerfish-staging-staging.s3.amazonaws.com/JXJTKHKE6NHH7ODZ5VYEIKQVQM.JPG",
      "version": "0.5.8",
      "width": 480
    },
    {
      "_id": "PP7ESJELPRCT5A2YJK74E67XDI",
      "type": "text",
      "content": "My favorite animal is the armadillo."
    }
  ]
```

But in the published edition of the story, we have this:

`https://api.thepost.arcpublishing.com/content/v3/stories?_id=TLAWPF3RHJAW5LWWJB2DHQXDT4&published=true`
```json

  "content_elements": [
    {
      "_id": "QOMQZYXMXVDEDIHROL2D3EUSHA",
      "type": "text",
      "content": "This document was created via a call to the Story API."
    },
    {
      "_id": "IA6RGMNIIVF6FCOFTQT22FFRJ4",
      "type": "reference",
      "referent": {
        "type": "image",
        "id": "JXJTKHKE6NHH7ODZ5VYEIKQVQM",
        "provider": ""
      }
    },
    {
      "_id": "PP7ESJELPRCT5A2YJK74E67XDI",
      "type": "text",
      "content": "My favorite animal is the armadillo."
    }
  ]

```


The image we embedded in our story doesn't inflate at all in the published edition. The reason for this is that, in Arc, images have their own publish status *independent of the embedding story*. This means that images can be published or unpublished without manually updating every story that image ever appeared in. (Incidentally, the same is true for galleries and video.)

This feature is convenient when you want to remove a photo everywhere, but it means we need to take one more step for our story to be truly finished -- we need to publish the image itself in the Image API.

Retrieve the saved photo document:
```
curl -X GET https://api.thepost.arcpublishing.com/photo/v2/photos/JXJTKHKE6NHH7ODZ5VYEIKQVQM

{"_id":"JXJTKHKE6NHH7ODZ5VYEIKQVQM","additional_properties":{"galleries":[],"mime_type":"image/jpeg","originalUrl":"https://arc-anglerfish-staging-staging.s3.amazonaws.com/JXJTKHKE6NHH7ODZ5VYEIKQVQM.JPG","proxyUrl":"/photo/resize/F2zV-JyTwVOqeRD6oUEAJQlaD-4=/arc-anglerfish-staging-staging/JXJTKHKE6NHH7ODZ5VYEIKQVQM.JPG","published":false,"resizeUrl":"http://anglerfish-staging-thumbor.internal.arc2.nile.works/F2zV-JyTwVOqeRD6oUEAJQlaD-4=/arc-anglerfish-staging-staging/JXJTKHKE6NHH7ODZ5VYEIKQVQM.JPG","version":0},"created_date":"2017-12-12T16:44:07+00:00","height":320,"last_updated_date":"2017-12-12T16:44:07+00:00","licensable":false,"owner":{"id":"staging","name":"Organization Name Override Goes Here"},"type":"image","url":"https://arc-anglerfish-staging-staging.s3.amazonaws.com/JXJTKHKE6NHH7ODZ5VYEIKQVQM.JPG","version":"0.5.8","width":480}
```

Change `additional_properties.published` to `true` and PUT the document back:
```
curl -X PUT https://api.thepost.arcpublishing.com/photo/v2/photos/JXJTKHKE6NHH7ODZ5VYEIKQVQM -d '{"_id":"JXJTKHKE6NHH7ODZ5VYEIKQVQM","additional_properties":{"galleries":[],"mime_type":"image/jpeg","originalUrl":"https://arc-anglerfish-staging-staging.s3.amazonaws.com/JXJTKHKE6NHH7ODZ5VYEIKQVQM.JPG","proxyUrl":"/photo/resize/F2zV-JyTwVOqeRD6oUEAJQlaD-4=/arc-anglerfish-staging-staging/JXJTKHKE6NHH7ODZ5VYEIKQVQM.JPG","published":true,"resizeUrl":"http://anglerfish-staging-thumbor.internal.arc2.nile.works/F2zV-JyTwVOqeRD6oUEAJQlaD-4=/arc-anglerfish-staging-staging/JXJTKHKE6NHH7ODZ5VYEIKQVQM.JPG","version":0},"created_date":"2017-12-12T16:44:07+00:00","height":320,"last_updated_date":"2017-12-12T16:44:07+00:00","licensable":false,"owner":{"id":"staging","name":"Organization Name Override Goes Here"},"type":"image","url":"https://arc-anglerfish-staging-staging.s3.amazonaws.com/JXJTKHKE6NHH7ODZ5VYEIKQVQM.JPG","version":"0.5.8","width":480}'

{"_id":"JXJTKHKE6NHH7ODZ5VYEIKQVQM","additional_properties":{"galleries":[],"mime_type":"image/jpeg","originalUrl":"https://arc-anglerfish-staging-staging.s3.amazonaws.com/public/JXJTKHKE6NHH7ODZ5VYEIKQVQM.JPG","published":true,"version":1},"created_date":"2017-12-12T16:44:07+00:00","height":320,"last_updated_date":"2017-12-12T20:40:36+00:00","licensable":false,"owner":{"id":"staging","name":"Organization Name Override Goes Here"},"type":"image","url":"https://arc-anglerfish-staging-staging.s3.amazonaws.com/public/JXJTKHKE6NHH7ODZ5VYEIKQVQM.JPG","version":"0.5.8","width":480}
```

Now if we check back in with our published story, we should see the inflated image present:

```
curl -X GET 'https://api.thepost.arcpublishing.com/content/v3/stories?_id=TLAWPF3RHJAW5LWWJB2DHQXDT4&published=true'

{"_id":"TLAWPF3RHJAW5LWWJB2DHQXDT4","type":"story","version":"0.5.8","content_elements":[{"_id":"QOMQZYXMXVDEDIHROL2D3EUSHA","type":"text","content":"This document was created via a call to the Story API."},{"_id":"JXJTKHKE6NHH7ODZ5VYEIKQVQM","additional_properties":{"galleries":[],"mime_type":"image/jpeg","originalUrl":"https://arc-anglerfish-staging-staging.s3.amazonaws.com/public/JXJTKHKE6NHH7ODZ5VYEIKQVQM.JPG","published":true,"version":1},"created_date":"2017-12-12T16:44:07+00:00","height":320,"last_updated_date":"2017-12-12T20:39:45+00:00","licensable":false,"owner":{"id":"staging","name":"Organization Name Override Goes Here"},"type":"image","url":"https://arc-anglerfish-staging-staging.s3.amazonaws.com/public/JXJTKHKE6NHH7ODZ5VYEIKQVQM.JPG","version":"0.5.8","width":480},{"_id":"PP7ESJELPRCT5A2YJK74E67XDI","type":"text","content":"My favorite animal is the armadillo."}],"created_date":"2017-12-11T19:52:12.736Z","revision":{"revision_id":"AQXPF7XEGFCQJOZLM3JPO3UO7A","parent_id":"BWSCXT3Z7ZE2ZMMP54MYRF45TU","editions":["default"],"branch":"default","published":true},"last_updated_date":"2017-12-12T17:07:01.334Z","headlines":{"basic":"My First Arc Document, Updated"},"owner":{"id":"staging"},"display_date":"2017-12-11T20:53:20.825Z","credits":{"by":[{"_id":"engelg2","type":"author","version":"0.5.8","name":"Gregory Engel","description":"A developer at Arc Publishing","additional_properties":{"original":{"_id":"engelg2","name":"Gregory Engel","bio":"A developer at Arc Publishing"}}}]},"subheadlines":{"basic":"Created in Arc"},"first_publish_date":"2017-12-11T20:53:20.825Z","taxonomy":{"sites":[{"_id":"/science","type":"site","version":"0.5.8","name":"Science","path":"/science","parent_id":"/","additional_properties":{"original":{"_id":"/science","name":"Science","parent":"/","inactive":false,"order":100059}}},{"_id":"/science/animals","type":"site","version":"0.5.8","name":"Animals","path":"/science/animals","parent_id":"/science","additional_properties":{"original":{"_id":"/science/animals","name":"Animals","parent":"/science","inactive":false,"order":100069}}}]},"additional_properties":{"has_published_copy":true},"publish_date":"2017-12-12T17:48:43.578Z","canonical_url":"/my-first-arc-document","publishing":{"scheduled_operations":{"publish_edition":[],"unpublish_edition":[]}}}
```
 ...and indeed we do!

## Appendix: Inflation Details

### General Notes & Limitations

In most cases, inflation will only ever be performed to a depth of 1. References on a source document added to the Content API will be resolved, but references on the referent objects (those added to the document) will not be inflated. The only exception to this is that after images have been inflated, author references on the inflated images will be resolved.

*The maximum number of references on a source document is 300.* Documents with a number of first-depth references greater than this limit will be rejected.

### Authors

Authors are inflated from the Author API and can be inflated only within the `credits` object:

```json
{
  "type": "story",
  "version": "0.5.8",

  "credits": {
    "by": [
      {
        "type": "reference",
        "referent": {
          "type": "author",
          "id": "engelg",
          "provider": ""
        }
      },
      {
        "type": "reference",
        "referent": {
          "type": "author",
          "id": "kimt",
          "provider": ""
        }
      }
    ],
    "photos_by": [
      {
        "type": "reference",
        "referent": {
          "type": "author",
          "id": "burnettj",
          "provider": ""
        }
      }
    ]
  }
}
```

### Sections

Sections are inflated from the Site API and can be inflated only in `taxonomy.sites` and `taxonomy.primary_site`:

```json
{
  "type": "story",
  "version": "0.5.8",

  "taxonomy": {
    "primary_site": {
      "type": "reference",
      "referent": {
        "type": "site",
        "id": "/sports",
        "provider": ""
      }
    },
    "sites": [
      {
        "type": "reference",
        "referent": {
          "type": "site",
          "id": "/sports",
          "provider": ""
        }
      },
      {
        "type": "reference",
        "referent": {
          "type": "site",
          "id": "/sports/lakers",
          "provider": ""
        }
      }
    ]
  }
}
```

### Images

Images are inflated from the Photo API and can be inflated in `content_elements`, in `related_content`, and in `promo_items`.

* `content_elements` is an ordered list of the core content of a document.
* `related_content` is a map of arbitrary keys to lists of references.
* `promo_items` is a map of arbitrary keys to a single reference.

```json

{
  "type": "story",
  "version": "0.5.8",

  "content_elements": [
    {
      "type": "text",
      "content": "An image follows this paragraph."
    },
    {
      "type": "reference",
      "referent": {
        "type": "image",
        "id": "ABCDEFGHIJKLMNOPQRSTUVWXYZ",
        "provider": ""
      }
    }
  ],

  "related_content": {
    "basic": [
      {
        "type": "reference",
        "referent": {
          "type": "image",
          "id": "AAAAAAAAAAAAAAAAAAAAAAAAAA",
          "provider": ""
        }
      }
    ]
  },

  "promo_items": {
    "basic": {
      "type": "reference",
      "referent": {
        "type": "image",
        "id": "BBBBBBBBBBBBBBBBBBBBBBBBBB",
        "provider": ""
      }
    }
  }
}
```

### Other Documents

The three top-level types of document in the Content API *(Story, Video, and Gallery)* can themselves be inflated from another document. The placement rules for these are essentially the same as for images.

* Stories are inflated from the Story API
* Galleries are inflated from the Photo API / Anglerfish
* Videos are inflated from the Video API / Goldfish

```json
{
  "type": "story",
  "version": "0.5.8",

  "content_elements": [
    {
      "type": "text",
      "content": "An image follows this paragraph."
    },
    {
      "type": "reference",
      "referent": {
        "type": "story",
        "id": "DEFGHIJKLMNOPQRSTUVWXYZABC",
        "provider": ""
      }
    }
  ],

  "related_content": {
    "basic": [
      {
        "type": "reference",
        "referent": {
          "type": "video",
          "id": "AAAAAAAAAAAAAAAAAAAAAAABCD",
          "provider": ""
        }
      }
    ]
  },

  "promo_items": {
    "basic": {
      "type": "reference",
      "referent": {
        "type": "gallery",
        "id": "BBBBBBBBBBBBBBBBBBBBBBBCDE",
        "provider": ""
      }
    }
  }
}
```

### URLs

URLs are denormalized from the URL API. No reference is required -- the URL will be fetched by the document ID and placed in `canonical_url`. Note that despite the field name, the url has a relative path.


## Appendix: Additional Resources

The complete list of reference documentation for the APIs used in this document:

* [ANS Schema Reference](https://github.com/washingtonpost/ans-schema)
* [Content API](https://arcpublishing.atlassian.net/wiki/spaces/CA/pages/50928390/Content+API)
* [Story API](https://arcpublishing.atlassian.net/wiki/spaces/CA/pages/13338279/Story+API)
* [Author API](https://arcpublishing.atlassian.net/wiki/spaces/CA/pages/39157865/Author+Service+API)
* [Sites API](https://arcpublishing.atlassian.net/wiki/spaces/CA/pages/39157865/Author+Service+API)
* [Image API / Anglerfish](https://arcpublishing.atlassian.net/wiki/spaces/ANG/pages/13338195/Anglerfish+API)
* [Video API / Goldfish](https://arcpublishing.atlassian.net/wiki/spaces/GOL/pages/67895547/Video+API)
* [URL API](https://arcpublishing.atlassian.net/wiki/spaces/CA/pages/13338275/Url+Service)

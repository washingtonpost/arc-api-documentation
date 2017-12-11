# Example: Publishing a Document in Content API

## Goal

To store custom content in an ANS document in the Content API in a searchable manner.

## Prerequisites

* Knowledge of HTTP and cURL (or other HTTP client)
* JSON editor
* An active Arc account and basic auth credentials

## Procedure

### Accessing the APIs

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


### Creating your first document

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



### Searching for the document in Content API.

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



### Publishing a document

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


### Creating an Author

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

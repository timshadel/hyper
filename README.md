# Hyper — a JSON-based media type

`hyper` can be used to represent graphs of objects. It has links, [RFC 5988 rels][web-linking], and more.

[web-linking]: http://tools.ietf.org/html/rfc5988

## Basics

At its simplest, it holds data exactly like JSON.

```json
{
  "name": { "first": "Joe", "last": "Smith" }
}
```

Multiple values are put into an array—again, it's just JSON.

```json
{
  "name": "Bhavesh",
  "cars": [ { "name": "Camero" }, { "name": "Porsche" } ]
}
```

### Links

Objects may include links. Any object with an `href` key can be retrieved by following that
hypermedia reference. Consequently, the `href` also defines a new scope for interpreting
other tags, as we'll see when we get to [Queries](#query_templates)

```json
{
  "name": "Bhavesh",
  "employer": { "href": "/employers/acme" }
}
```

Usually you'll want more information than a link, but it's just custom data.

```json
{
  "name": "Bhavesh",
  "employer": {
    "href": "/employers/acme",
    "name": "Acme, Inc."
  }
}
```

The same interpretation rules apply when `href` is found on the root object. In this case, 
`href` is equivalent to `self` from RFC 5988, but without the stupid name.

```json
{
  "href": "/bhavesh",
  "name": "Bhavesh",
  "employer": {
    "href": "/employers/acme",
    "name": "Acme, Inc."
  }
}
```

### Composites

We can instruct the agent to include other resources in the final representation of this
hypermedia document by using the `src` key.

```json
{
  "href": "/bhavesh",
  "name": "Bhavesh",
  "employer": { "src": "/employers/acme" }
}
```

After reading this document, the agent would request `/employers/acme`, and substitute its
content in place of the object containing `src`.

```console
GET /employers/acme HTTP/1.1
Host: example.com
Accept: application/json, application/hyper+json

200 OK
Content-Type: application/hyper+json

{"href":"/employers/acme","name": "Acme, Inc.","website": "http://acme.com/"}
```

After retrieval:

```json
{
  "href": "/bhavesh",
  "name": "Bhavesh",
  "employer": {
    "href": "/employers/acme",
    "name": "Acme, Inc.",
    "website": "http://acme.com/"
  }
}
```

The agent will actually be giving the client an in-memory data structure similar to a DOM.
This means that we can use `src` to include other kinds of media types. If they are
understood, the agent will retrieve them and provide an appropriate representation of them
in the "DOM". The `type` key is used to hint at what the media of the resource may be, though
the server has ultimate control, and this is just a hint.

```json
{
  "href": "/bhavesh",
  "name": "Bhavesh",
  "picture": { "src": "/bhavesh/pic", "type": "image/jpeg" }
}
```

```console
GET /bhavesh/pic HTTP/1.1
Host: example.com
Accept: application/json, application/hyper+json, image/*

200 OK
Content-Type: image/png

...image data...
```

For example, if the client is on iOS, the agent may decide to create a `UIImage` object as
part of the "DOM" given to the client application layer.

```objective-c
@{
  @"href": @"/bhavesh",
  @"name": @"Bhavesh",
  @"picture": [UIImage imageWithData:(...image data...)]
}
```


### Embedded Lists & Rels

When a list has a hypermedia identity, you use objects with a special structure.

```json
{
  "name": "Bhavesh",
  "friends": {
    "href": "/bhavesh/friends",
    "data": [
      { "href": "/joe", "name": "Joe" },
      { "href": "/2342927", "name": "Bill" }
    ]
  }
}
```

When an object uses the reserved key `data`, then you may also use standard `rel` values
keys. This object has a paginated list, and the `next` link is interpreted using the
standard semantics defined in [Web Linking][web-linking].

```json
{
  "name": "Bhavesh",
  "friends": {
    "href": "/bhavesh/friends",
    "data": [
      { "href": "/joe", "name": "Joe" },
      { "href": "/2342927", "name": "Bill" }
    ],
    "next": "/bhavesh/friends?page=2"
  }
}
```

You can even do the same thing with objects which require metadata. Here we add a link
to `edit` the embedded `employer` object.

```json
{
  "href": "/bhavesh",
  "name": "Bhavesh",
  "employer": {
    "edit": "/employers/acme/edit",
    "data": {
      "href": "/employers/acme",
      "name": "Acme, Inc."
    }
  }
}
```

You could do the same thing with the main object.

```json
{
  "describedby": "/spec/profile",
  "hub": "/bhavesh/hub",
  "data": {
    "href": "/bhavesh",
    "name": "Bhavesh",
    "friends": {
      "href": "/bhavesh/friends",
      "data": [
        { "href": "/joe", "name": "Joe" },
        { "href": "/2342927", "name": "Bill" }
      ],
      "next": "/bhavesh/friends?page=2"
    }
  }
}
```

And remember, regular arrays are perfectly legal—they just don't have an independently
accessible representation.

```json
{
  "name": "Bhavesh",
  "favorite_colors": [ "franklin turquoise", "rosey rose" ],
  "friends": {
    "href": "/bhavesh/friends",
    "data": [
      { "href": "/joe", "name": "Joe" },
      { "href": "/2342927", "name": "Bill" }
    ],
    "next": "/bhavesh/friends?page=2"
  }
}
```

### Query Templates

Templates for possible queries are objects which have a `query` key. The `query` value
is interpreted with the [URI Template (RFC 6570)][uri-template]. Every `query` is
implied to query within the scope of the nearest `href`. In this example, the
`searchByFavoriteColor` query implies that it searches the list `/bhavesh/friends`.

[uri-template]: http://tools.ietf.org/html/rfc6570

```json
{
  "name": "Bhavesh",
  "favorite_colors": [ "franklin turquoise", "rosey rose" ],
  "friends": {
    "href": "/bhavesh/friends",
    "data": [
      { "href": "/joe", "name": "Joe" },
      { "href": "/2342927", "name": "Bill" }
    ],
    "next": "/bhavesh/friends?page=2",
    "searchByFavoriteColor": { "query": "/bhavesh/friends?favorite_color={colorName}" }
  }
}
```

If we wanted to define a search of all profiles we might do something like the following.
Here, `search.favoriteColor` doesn't have an implied scope, whereas
`data.friends.searchByFavoriteColor` still will only be searching within its immediate
ancestor `href` list, `/bhavesh/friends`.

```json
{
  "describedby": "/spec/profile",
  "hub": "/bhavesh/hub",
  "search": {
    "favoriteColor": { "query": "/profiles?favorite_color={colorName}" },
    "name": { "query": "/profiles?name={name}" },
    "username": { "query": "/{username}" }
  }
  "data": {
    "href": "/bhavesh",
    "name": "Bhavesh",
    "friends": {
      "href": "/bhavesh/friends",
      "data": [
        { "href": "/joe", "name": "Joe" },
        { "href": "/2342927", "name": "Bill" }
      ],
      "next": "/bhavesh/friends?page=2",
      "searchByFavoriteColor": { "query": "/bhavesh/friends?favorite_color={colorName}" }
    }
  }
}
```

### Actions

Showing agents how to take actions is really straightforward.

Here's a registration action. Like HTML forms, the `action` key is the URI where the action
will be submitted. The `method` key is used to indicate the HTTP verb used, and the `input`
key defines the structure of the request body, which will be `application/hyper+json`. If the
value of an input key is just a string, then it is the default value of that input, akin to
HTML's `hidden` input type.

```json
{
  "register": {
    "action": "/register",
    "method": "POST",
    "input": {
      "name": { "type": "text" },
      "email": { "type": "text" },
      "csrf_key": "a13fa7980eec29e4067623259d8012df365825ea9b8bca68db523427adb88eb8"
    }
  }
}
```

Here's the kind of request that we'd expect a user agent to submit for a registration.

```console
POST /register HTTP/1.1
Host: example.com
Content-Type: application/hyper+json

{"email":"invalid.email-address","csrf_key":"a13fa7980eec29e4067623259d8012df365825ea9b8bca68db523427adb88eb8"}
```

The input submitted is a conforming JSON document, with no keys besides `name`, `email`, or
'csrf_key', all of which contain text.

In order to require a value, we'll expand the document to include the `required` key. We've
also use the `type` key again here to indicate (imaginary) media types each key is expected
to conform to. Adding a `value` key provides a default value for an input. Note that this is
*not* the same as placeholder text, and it should be submitted unless the agent intentionally
changes it. Adding `pattern` to the input tells the agent how to use pre-populated data in
order to create the resulting input. The pattern below indicates that email addresses from
`example.com` can be used. The `pattern` value is not binding, and should be considered
helpful, but not relied upon to guarantee clean input.

```json
{
  "register": {
    "action": "/register",
    "method": "POST",
    "input": {
      "name": {
        "type": "text",
        "required": true,
        "value": "John Smith"
      },
      "email": {
        "type": "text/x-email",
        "required": true,
        "pattern": { "{username}@example.com" }
      }
    }
  }
}
```

Nested object inputs may be described as well. By nesting an `input` key inside the
named input, we tell the agent of another object to send. Also note that the `href`
key inside the `input` is _not_ interpreted as the `href` of the `input`, but instead
it is just the name of a field that is sent to the server.

```json
{
  "register": {
    "action": "/register",
    "method": "POST",
    "input": {
      "name": {
        "type": "text",
        "required": true
      },
      "email": {
        "type": "text/x-email",
        "required": true
      },
      "facebook": {
        "input": {
          "id" : "text" ,
          "username" : "text",
          "href": "text/href"
        }
      }
    }
  }
}
```

Here a valid registration request may look like this:

```console
POST /register HTTP/1.1
Host: example.com
Content-Type: application/hyper+json

{
  "name": "John Smith"
  "email": "john.smith@example.com",
  "facebook": {
    "id": "2340723049823409832",
    "username": "john.smith",
    "href": "https://facebook.com/john.smith"
  }
}
```

## All Together

Here's how a Facebook Graph API result would look:

```json
{
  "href": "/700260353", 
  "name": "John Smith", 
  "first_name": "John", 
  "last_name": "Smith", 
  "link": "https://www.facebook.com/john.smith", 
  "username": "john.smith", 
  "hometown": {
    "href": "/632661010378778", 
    "name": "New York, New York"
  }, 
  "location": {
    "href": "/420688751127837", 
    "name": "Los Angeles, California"
  }, 
  "sports": [
    {
      "href": "/583047910860643", 
      "name": "Football"
    }
  ], 
  "gender": "male", 
  "relationship_status": "Married", 
  "significant_other": {
    "name": "Jane Smith", 
    "href": "/8427207141"
  }, 
  "timezone": -8, 
  "locale": "en_US", 
  "languages": [
    {
      "href": "/227591371060595", 
      "name": "English"
    }, 
    {
      "href": "/253834810822491", 
      "name": "French"
    }
  ], 
  "friends": {
    "href": "/700260353/friends",
    "data": [
      {
        "name": "James Brown",
        "href": "/993704"
      },
      {
        "name": "Jeremy Bliss",
        "href": "/6842230"
      },
      {
        "name": "Ben Space",
        "href": "/8612905"
      },
      {
        "name": "Sarah Plain",
        "href": "/1676206"
      }
    ], 
    "next": "/700260353/friends?limit=10&offset=5"
  },
  "verified": true, 
  "updated_time": "2013-02-17T07:49:06+0000"
}
```

Here's an alternate version which includes actions

```json
{
  "hub": {
    "action": "/app23792387420/subscriptions",
    "method": "POST",
    "input": {
      "object": {
        "select": ["user", "page", "permissions"]
      },
      "fields": "array",
      "callback_url": "text/href",
      "verify_token": "text"
    }
  },
  "data": {
    "href": "/700260353", 
    "name": "John Smith", 
    "first_name": "John", 
    "last_name": "Smith", 
    "link": "https://www.facebook.com/john.smith", 
    "username": "john.smith", 
    "hometown": {
      "href": "/632661010378778", 
      "name": "New York, New York"
    }, 
    "location": {
      "href": "/420688751127837", 
      "name": "Los Angeles, California"
    }, 
    "sports": [
      {
        "href": "/583047910860643", 
        "name": "Football"
      }
    ], 
    "gender": "male", 
    "relationship_status": "Married", 
    "significant_other": {
      "name": "Jane Smith", 
      "href": "/8427207141"
    }, 
    "timezone": -8, 
    "locale": "en_US", 
    "languages": [
      {
        "href": "/227591371060595", 
        "name": "English"
      }, 
      {
        "href": "/253834810822491", 
        "name": "French"
      }
    ], 
    "friends": { "src": "/700260353/friends" },
    "verified": true, 
    "updated_time": "2013-02-17T07:49:06+0000"
  }
}
```

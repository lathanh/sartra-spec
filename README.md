Abstract
-------------------------------------------------------------------------------
This document specifies a language for HTTP clients to tell an HTTP service, in
its request, about additional requests that it would make based on the
response(s) so the service can perform them instead, and a format for the
service to include the multiple resources along with the main response(s) in a
single multipart payload, reducing latency for the client.

Introduction
-------------------------------------------------------------------------------
For a normalized, RESTful API — one where object resources are served
individually rather than bundled together, tailored for the use case —
retrieving all of the objects necessary to display a UI could take several round
trips.
A first request may reference other objects, and only with the response to the
first request can subsequent requests be made to fetch those referenced objects
(and perhaps further requests may be needed to fetch referenced related objects).
Each request takes time to make it to the server, sometimes just a handful of
milliseconds, but possibly hundreds of milliseconds over slower connections
and/or geographically distant clients.
If the server can be told that objects referenced in a response are also needed,
the server can also send those proactively, eliminating some of the latency of
additional round-trips.

### Summary
This specification, the SARTRA (or Server-Assisted Round-Trip Reducing
Aggregation) protocol, defines a language for clients to tell a server about
resources referenced in a response — which the server should also fetch — and
a format for the server to include those objects in a single, multipart MIME
payload.
The responses are structured data (such as XML or JSON) and the client has
knowledge of the response's schema, so it can describe where other resource
references can be found.
The SARTRA server looks for the referenced resources, and fetches those
resources as well (performs the round-trips on behalf of the client).
Because the server is closer to the servers of those resources, the round-trips
are significantly less costly (e.g., possibly less than a millisecond instead of
tens or hundreds of milliseconds).
Resources can be referenced by URL/URI, or anything that the server is able to
resolve.

It is like [Batching](https://tools.ietf.org/id/draft-snell-http-batch-00.html)
in that one request and response are used to contain many requests and
responses.
In fact, it builds upon the Batch format, allowing multiple requests in each
round-trip, and in the absence of directives about related objects, it behaves
just like batching.

It is distinct from
[Pipelining](https://www.w3.org/Protocols/rfc2616/rfc2616-sec8.html), where the
client doesn't need to wait for (and parse) a response before it makes
additional requests (those additional requests are not resources referenced and
discovered in the prior response).

### Client-side Architectural Impact
In order to reduce round-trips, the client must provide information about what
resources it would need — that it would have explicitly requested in subsequent
requests — in the initial request, or "RTR" ("round-trip reduction") data.
A client-side RTR service will use the SARTRA protocol (and remote SARTRA
service) to retrieve all of the resources in just one request.
Upon completion of the RTR request, the consumer then expects that all of the
resources are provided (e.g., in a return object) or available (e.g.,
retrievable from a local database).

The RTR service on clients also has the flexibility to just simplify/abstract
round-trips and not resort to the SARTRA protocol.
See [Client-side flexibility](#Client-side flexibility) in the Appendix.

### Example Scenario
For this example, a mobile application wants to show the user the list of
Messages in their Inbox. And in this example:
  *  There are three messages, and showing each of the Messages in the list also
     requires:
     * The User who sent the Message which includes the person's name and a URL
       to their photo
  * Those three messages were sent by two unique users
  * The server and client use JSON (as the data transport format)
  * Each object is identified by a globally-unique URI, and the server knows how
    to fetch an object given (just) its URI.

#### 1st Request
**Request**: `GET /mailbox/Inbox`

**Response**:
```
{
  "uri" : "/mailbox/Inbox",
  "messages" : [
    { "messageUri" : "/message/1" },
    { "messageUri" : "/message/99" },
    { "messageUri" : "/message/123" },
  ]
}
```

### 2nd Request
Now the client will fetch the individual Message resources (with Batching, all
three of these requests could be made in one, multipart HTTP request).
One of those requests and responses will look like this:

**Request**: `GET /message/1`

**Response**:
```
{
  "uri" : "/message/1",
  "senderUri" : "/user/1337",
  "sendTime" : "Oct 22, 2014 23:16 PM PDT",
  "subject" : "Hello World",
  "body" : "foo"
}
```

### 3rd Request
The client has received the three Messages, collected the `senderUris` of each
message (and they're two unique Users), and now needs to fetch the two users.
One of those requests and responses may look like this:

**Request**: `GET /user/1337`

**Response:**
```
{
  "uri" : "/user/1337",
  "firstName" : "Tony",
  "photos" : {
    "thumbnailUrl" : "http://example.com/photos/1337_thumb.png"
  }
}
```

Over three round-trips we've requested 6 resources (1 list, 3 messages, and
2 users) needed to show the Inbox.

Basic Specification
-------------------------------------------------------------------------------
The client communicates with a SARTRA HTTP endpoint, using a multipart request
format to allow the client to include round-trip-reduction information ("RTR
Spec") to the server along with the base request.
The server will use the RTR Spec to collect the URIs from the response of the
base request, fetch the resources at the collected URIs (repeating if nested
was defined), and aggregate all of the objects into one response.

This is built on top of multipart/batch with two additions:
  1. Each batch part may also contain a "sartra" part, which describes where to
     look in the response for more resource URIs to fetch
  2. The response may contain parts for objects that were not explicitly
     requested.

### Endpoint
The endpoint that the client hits is dedicated to SARTRA, whether part of an
existing server (e.g., https://api.example.com/sartra), or its own server
(e.g., https://sartra.example.com/).

### Header and Parameters
The `Content-Type` header is used to indicate to the server that the request
body will use the SARTRA format, and provide parameters.

  * `Content-Type: multipart/sartra;` — SARTRA Content-Type
  * `type="application/http;version=1.1";` — Each part is itself an HTTP
    request
  * `sartra-boundary=sartra` — Specifies the token that will be used as
    delimiter between the request itself from the SARTRA part.
    It follows the
    [Multipart common syntax](https://www.w3.org/Protocols/rfc1341/7_2_Multipart.html)
    (for example, it must be no longer than 70 characters, and the use of two
    hyphens when used as boundaries).
  * `batch-boundary=batch` — Specifies the token that will be used as the
    delimiter between separate requests.

#### Example
```
Content-Type: multipart/sartra;
  type="application/http;version=1.1";
  sartra-boundary=sartra;
  batch-boundary=batch
```

### Request Body: Multipart with RTR Spec
The SARTRA request is multipart, allowing multiple explicit requests for
objects (like Batching).
Each part consists of an embedded HTTP request, and optionally a sub-part
containing the RTR Spec that tells the server how to find resource references
within the response of that request.

Immediately following a `batch-boundary` separator (that is, `--` followed by
the batch-boundary value) is an HTTP part, which consists of:
  1. Part headers, including:
     * "`Content-Type: application/http;version=1.1`": which indicates that the
       part is an HTTP request
     * `Content-ID`: Specifying a reference identifier for the HTTP request
       message.
  2. A blank line
  3. The HTTP request (namely: request line, headers, and optionally body)

The HTTP request may then be followed by the `sartra-boundary`.
If so, what follows is the RTR Spec which tells the server how to find resource
URIs in a response, which it should also fetch and include in the multipart
response.
The RTR Spec, by default, is a JSON object with the following structure:

  * _Array_: allowing resources to be found in multiple parts of the response
    * _Hash_: for each path to search. Has the following structure:
      * "`label`": (_optional_) A client-specified label that will be included
        in the response, allowing the client to identify why a resource was
        included in the SARTRA response.
        It must consist only of alphanumeric, '-' (hyphen), or '_' (underscore)
        characters.
      * "`path-lang`": (_optional_) The language that the `path` is in, which
        the client and server pre-determined is supported by the server.
        This allows for more powerful path specs ()such as regular expressions),
        or to represent resource locations in proprietary response structures.
        The default language (if a value is not specified) is "jsonpath".
        Values other than "jsonpath" should be prefixed with "x-" in order to
        indicate that the path-lang is server/vendor-specific.
      * "`path`": A specification for where to find resource URIs in the
        response.
      * "`rtr`": Short for "round-trip reduction," a nested RTR Spec (thus an
        array), which allows URIs/resources to be found in the bodies of
        resources found by the outer RTR.
        `rtr` can be nested arbitrarily deep.

Following the part (HTTP request or HTTP request with RTR Spec) must be either
another batch-boundary separator and another part, or the final batch-boundary
(that is, the batch-boundary value preceded and followed by '--').


#### Example
```
POST /sartra HTTP/1.1
Host: example.org
Content-Type: multipart/sartra;
  type="application/http;version=1.1";
  sartra-boundary=sartra
  batch-boundary=batch
Mime-Version: 1.0

--batch
Content-Type: application/http;version=1.1
Content-Transfer-Encoding: binary
Content-ID: <mailbox-inbox@example.org>

GET /mailbox/Inbox HTTP/1.1
Host: example.org

--sartra
[
  { "label" : "messages",
    "path" : "messages[]/messageUri",
    "rtr" : [
      { "label" : "senders",
        "path-lang" : "jsonpath",
        "path" : "senderUri"
      }
    ]
  }
]
--batch--
```

In this example:
  * "`path`": The server should find an element named "messages", which is an
    array.
    For each of the objects of that array, collect the URI under the attribute
    "messageUri"
  * "`rtr`" : The server should look in each of the messages' "senderUri"
    attribute for more URIs of objects to fetch

### Response
Each resource retrieved by the SARTRA service is included linearly as a part in
a multipart response, not necessarily in a particular order.

  * "`In-Reply-To`" If the resource was directly requested, the Content-ID of
    that request.
  * "`X-Sartra`" If the resource was requested as a result of being referenced
    via a 'sarta' spec, the label, in quotes, of that spec, followed by the
    Content-ID of the main request.

#### Example
```
HTTP/1.1 200 OK
Server: example.org
Content-Type: multipart/sartra;
  type="application/http;type=1.1";
  boundary=PartBoundary
Mime-Version: 1.0

--PartBoundary
Content-Type: application/http;version=1.1
Content-Transfer-Encoding: binary
In-Reply-To: <mailbox-inbox@example.org>

HTTP/1.1 200 OK
Server: example.org

{
  "uri" : "/mailbox/Inbox",
  "messages" : [
    { "messageUri" : "/message/1" },
    { "messageUri" : "/message/99" },
    { "messageUri" : "/message/123" },
  ]
}

--PartBoundary
Content-Type: application/http;version=1.1
Content-Transfer-Encoding: binary
X-Sartra: "messages" <mailbox-inbox@example.org>

HTTP/1.1 200 OK
Server: example.org

{
  "uri" : "/message/99",
  "senderUri" : "/user/1337",
  "sendTime" : "Oct 22, 2014 11:11 PM PDT",
  "subject" : "Hello Again",
  "body" : "bar"
}

--PartBoundary
Content-Type: application/http;version=1.1
Content-Transfer-Encoding: binary
X-Sartra: "messages" <mailbox-inbox@example.org>

HTTP/1.1 200 OK
Server: example.org

{
  "uri" : "/message/123",
  "senderUri" : "/user/321",
  "sendTime" : "Oct 23, 2014 1:23 PM PDT",
  "subject" : "Test",
  "body" : "This is a test message"
}

--PartBoundary
Content-Type: application/http;version=1.1
Content-Transfer-Encoding: binary
X-Sartra: "messages" <mailbox-inbox@example.org>

HTTP/1.1 200 OK
Server: example.org

{
  "uri" : "/message/1",
  "senderUri" : "/user/1337",
  "sendTime" : "Oct 24, 2014 10:10 PM PDT",
  "subject" : "Hello again",
  "body" : "Are you ignoring me?"
}

--PartBoundary
Content-Type: application/http;version=1.1
Content-Transfer-Encoding: binary
X-Sartra: messages/sender <mailbox-inbox@example.org>

HTTP/1.1 200 OK
Server: example.org

{
  "uri" : "/user/1337",
  "firstName" : "Tony",
  "photos" : {
    "thumbnailUrl" : "http://example.com/photos/1337_thumb.png"
  }
}

--PartBoundary
Content-Type: application/http;version=1.1
Content-Transfer-Encoding: binary
X-Sartra: messages/sender <mailbox-inbox@example.org>

HTTP/1.1 200 OK
Server: example.org

{
  "uri" : "/user/321",
  "firstName" : "Ashley",
  "photos" : {
    "thumbnailUrl" : "http://example.com/photos/321_thumb.png"
  }
}

--PartBoundary--
```

This example multipart response contains:
  * The list of messages in the Inbox
  * The three messages referenced by the Inbox
  * The two unique users referenced by those three messages

#### Notes
##### Ordering
This spec does not require that resources be returned in any particular order.
For efficiency, a service may append resources to the response in the order they
are received by the data services, which may not necessarily be the order in
which they were requested.
SARTRA service implementations may choose to offer ordering through a custom
header parameter.

Appendix
===============================================================================

Benefits and Drawbacks
-------------------------------------------------------------------------------
### Compression
Because more resources are included in the body of a response, that response is
more compressible (compression generally yields a higher ratio for text as it
gets larger, especially for structured data that are similar to each other).

### De-duplication
The SARTRA service has the ability to keep track of all resources referenced and
request them only once and return them in the response only once.

### Cached Data
Clients are likely to cache resources so that they don't need to be re-fetched
when needed again within a short period of time.
However, a SARTRA service will fetch and return objects not knowing whether the
client already has it in its cache.
Depending on use patterns, which varies among different apps and users,
the resources that a SARTRA service sends to the client that it already had may
have a higher cost (viz., additional latency) than benefit.

There could be ways to mitigate this; for example, the client could send a list
of URIs of some of the resources it already has that could likely be referenced.

Additional Considerations
-------------------------------------------------------------------------------
#### Client-side Flexibility
The round-trip reduction (RTR) service on clients — clients which could be
mobile phones apps or other server-side services — is first simply an
abstraction (interface) of some implementation/service that aggregates
round-trips.
That is, the programming API simply takes in request objects along with
specification for what to fetch in subsequent round-trips, and returns all of
the requested objects without necessarily any claim on how exactly that will be
accomplished.
This abstraction provides the flexibility for the implementation to perform RTR
in a way most appropriate for the situation, sometimes even without the actual
SARTRA protocol.
For example, a webapp that is physically near the services it communicates with
could choose to use an RTR implementation that locally (within the same app)
performs all of the round-trips.
In this case, consumers simply benefit from the convenience of getting all the
data it needs in just one request.
The abstraction further affords the flexibility that the implementation may
change or add additional complexity (e.g., using SARTRA for some data, and just
performing round-trips itself locally for other data) without any change to the
consumer.

FAQ
-------------------------------------------------------------------------------
### Does this work with all data formats?
It can work with any format as long as both the client and server agree on the
path syntax and the server is able to parse the response and apply the path to
it.

This specification defines the path syntax for JSON and XML, but allows the
client to specify a path syntax — including vendor-specific syntax — that the
server understands.

Further, the path-lang doesn't need to have a specific relationship with the
content-type of the resource.
For example, "jsonpath" can be used to specify the path to URIs in an XML
response (as both have a hierarchical structure).

### Can SARTRA perform any logic on what URI to request; for example, generating a URL from part of a reference URL
Yes.
The default path-lang ("jsonpath") does not support this, but "jsonpath-regexp"
does, and servers can also support custom path-langs with any functionality
it wishes to add.

Miscellaneous
-------------------------------------------------------------------------------
### Aggregation vs Algorithm
The need to make multiple round-trips in order to fetch data can be organized
into two categories:

  1. **Algorithm**: Determining what to show, or what data to return.
     Criteria might include choosing a subset of data based on text matching, or
     sorting the data.
  2. **Display, Use-case**: Once it has been determined what data to show, more
     data may needed for that 

#### Aggregation for Algorithms
For example, sorting a user's friend list would require two round-trips:
   1. Fetch the list of UserIds of friends.
   2. Fetch the User object of all of those friends.

Then those friends can be sorted, and then the UserIds of those friends can be
returned to the requestor (a subset probably, such as the first 100).

Algorithms such as this could benefit under the hood from aggregation.
That is, the ability to fetch data that would require multiple round-trips more
efficiently could make the algorithm take less time to perform.

However, once the algorithm is complete, it should also only return the minimum
amount of data as possible that represents the result of the algorithm.
In the example above, that means only returning the list of UserIds of the
friends (in the desired sort order).
This:

  1. Is often to keep the algorithm from return data it is not responsible for.
     In this example, the service managing the friend list doesn't return User
     objects that are the responsibility of the User service.
  2. Keeps the algorithm separate from use-cases, minimizing its coupling to
     callers (maximizing its flexibility).
     In other words, the client might, for example, simply want to have the
     UserIds of the friends, not needing the User objects themselves, and this
     allows the client to determine that behavior.
  3. Allows the client to take advantage of caching (e.g., not requesting Users
     it already has in its cache/database.
  4. Makes it easier for the service to scale more easily since its jobs are
     more separated and simpler.
  5. Allows a decoupled, flexible solution like SARTRA to help make use-cases
     more efficient.

#### Aggregation for Display, Use-case
Continuing the example above, let's say that along with each friend, our client
wants to also show the time and subject of the last message sent to or received
from that friend.

So, after the algorithm has sorted the friends and determined the first 100 to
show, the client will also need to be provided with the last message.
If, in an effort to minimize round-trips that the client needs to make, we were
to include the last message as well as the list of friends (and the User object
of each of the friends) in the algorithm API's response, the API endpoint and
response would become tightly coupled with the client's use case.
When another use case comes up, either the API needs to be updated -- increasing
complexity of that API and probably introducing inefficiencies (because the
response is likely to contain data that the other use-case doesn't need) -- or
a new API created (also increasing complexity of the service).

Instead, with SARTRA performing the aggregation of all the data the client needs
to support the use-case, the client gains the benefit of round-trip reduction
while keeping the use-case decoupled from the algorithm API and allowing the
algorithm to remain simple (with the benefits that simplicity brings, such as
better maintainability and scalability).

Copyright Notice
===============================================================================
Copyright © Robert LaThanh

This work is licensed under the Creative Commons
Attribution-NonCommercial-NoDerivatives 4.0 International License.
To view a copy of this license, visit
http://creativecommons.org/licenses/by-nc-nd/4.0/ or send a letter to
Creative Commons, PO Box 1866, Mountain View, CA 94042, USA.

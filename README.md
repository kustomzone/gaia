Gaia: A decentralized high-performance storage system
====================================

This document describes the high-level design and implementation of the
Gaia storage system. It includes specifications for backend storage drivers
and interactions between developer APIs and the Gaia service. 

Developers who wish to _use_ the Gaia storage system should see the
`blockstack.js` APIs
documented [here](https://github.com/blockstack/blockstack.js/tree/master/src/storage) and [here](http://blockstack.github.io/blockstack.js/#storage).

# Overview

<!--- I swapped these two paragraphs and did some word-smithing (jcn) -->
Gaia works by hosting data in one or more existing storage systems of the user's choice.
These storage systems are typically cloud storage systems.  We currently have driver support
for S3 and Azure Blob Storage, but the driver model allows for other backend support
as well. The point is, the user gets to choose where their data lives, and Gaia enables
applications to access it via a uniform API.

Blockstack applications use the Gaia storage system to store data on behalf of a user.
When the user logs in to an application, the authentication process gives the application
the URL of a Gaia hub, which performs writes on behalf of that user. The Gaia hub authenticates
writes to a location by requiring a valid authentication token, generated by a private key
authorized to write at that location.

# Write-to and Read-from URL Guarantees

A performance and simplicity oriented guarantee of the Gaia
specification is that when an application submits a _write_ to a URL
`https://myhub.service.org/store/foo/bar`, the application is guaranteed to
be able to read from a URL `https://myreads.com/foo/bar`. While the
_prefix_ of the read-from URL may change between the two, the suffix
must be the same as the write-to URL.

This allows an application to know _exactly_ where a written file can
be read from, given the read prefix. To obtain that read prefix,
the Gaia service defines an endpoint:

```
GET /hub_info/
```

which returns a JSON object with a `read_url_prefix`.


For example, if my service returns:

```javascript
{ ...,
  "read_url_prefix": "https://myservice.org/read/"
}
```

I know that if I submit a write request to:

```
https://myservice.org/store/1DHvWDj834zPAkwMhpXdYbCYh4PomwQfzz/0/profile.json
```

That I will be able to read that file from:

```
https://myservice.org/read/1DHvWDj834zPAkwMhpXdYbCYh4PomwQfzz/0/profile.json
```


# Address-based Access-Control

Access control in a gaia storage hub is performed on a per-address
basis.  Writes to URLs `/store/<address>/<file>` are only allowed if
the writer can demonstrate that they control _that_ address. This is
achieved via an authentication token, which is a message _signed_ by
the private-key associated with that address. The message itself is a
challenge-text, returned via the `/hub_info/` endpoint.

In more detail, this signed message is:

```
BASE64({ "signature" : ECDSA_SIGN(SHA256(challenge-text)),
         "publickey" : PUBLICKEY_HEX })
```

Currently, challenge-text must match the _known_ challenge-text on
the gaia storage hub. However, as future work enables more extensible
forms of authentication, we could extend this to allow the auth
token to include the challenge-text as well, which the gaia storage hub
would then need to also validate.


# Data storage format

A gaia storage hub will store the written data _exactly_ as
given. This means that the storage hub _does not_ provide many
different kinds of guarantees about the data. It does not ensure that
data is validly formatted, contains valid signatures, or is
encrypted. Rather, the design philosophy is that these concerns are
client-side concerns. Client libraries (such as `blockstack.js`) are
capable of providing these guarantees, and we use a liberal definition of
the end-to-end principle to guide this design decision.

# Operation of a Gaia Hub

## Configuration files

A configuration JSON file should be stored either in the top-level directory
of the hub server, or a file location may be specified in the environment
variable `CONFIG_PATH`.

An example configuration file is provided in (./hub/config.sample.json)
You can specify the logging level, the number of social proofs required
for addresses to write to the system, the backend driver, the credentials
for that backend driver, and the readURL for the storage provider.

### Private hubs

A private hub services requests for a single user. This is controlled
via _whitelisting_ the addresses allowed to write files. In order to
support application storage, because each application uses a different
app- and user-specific address, each application you wish to use must
be added to the whitelist separately. An extensible authentication
approach would obviate the need for this manual whitelisting, but that
remains as future work.

### Open-membership hubs

An open-membership hub will allow writes for _any_ address top-level directory,
each request will still be validated such that write requests must provide valid
authentication tokens for that address. Operating in this mode is recommended
for service and identity providers who wish to support many different users.

In order to limit the users that may interact with such a hub to users
who provide social proofs of identity, we support an execution mode
where the hub checks that a user's profile.json object contains
_social proofs_ in order to be able to write to other locations. This
can be configured via the [config.json](#configuration-files).

# Driver model

Gaia hub drivers are fairly simple. The biggest requirement is the ability
to fulfill the _write-to/read-from_ URL guarantee. As currently implemented
a gaia hub driver must implement the following two functions:

```javascript
/**
 * Performs the actual write of a file to `path`
 *   the file must be readable at `${getReadURLPrefix()}/${storageToplevel}/${path}`
 *
 * @param { String } options.path - path of the file.
 * @param { String } options.storageToplevel - the top level directory to store the file in
 * @param { String } options.contentType - the HTTP content-type of the file
 * @param { stream.Readable } options.stream - the data to be stored at `path`
 * @param { Integer } options.contentLength - the bytes of content in the stream
 * @returns { Promise } that resolves to the public-readable URL of the stored content.
 */


performWrite (options: { path, contentType,
                         stream, contentLength: Number })

/**
 * Return the prefix for reading files from.
 *  a write to the path `foo` should be readable from
 *  `${getReadURLPrefix()}foo`
 * @returns {String} the read url prefix.
 */
getReadURLPrefix ()
```

# HTTP API

The Gaia storage API defines only three endpoints.

```
GET ${read-url-prefix}/${address}/${path}
```

This returns the data stored by the gaia hub at `${path}`. In order
for this to be usable from web applications, this read path _must_
set the appropriate CORS headers. The HTTP Content-Type of the file
should match the Content-Type of the corresponding write.

```
POST ${hubUrl}/store/${address}/${path}
```

This performs a write to the gaia hub at `${path}`. 

On success, it returns a `202` status, and a JSON object:

```javascript
{
 "publicUrl": "${read-url-prefix}/${address}/${path}"
}
```

The `POST` must contain an authentication header with a bearer token.
The bearer token's content and generation is described in
the [access control](#address-based-access-control) section of this
document.

```
GET ${hubUrl}/hub_info/
```

Returns a JSON object:

```javascript
{
 "challenge_text": "text-which-must-be-signed-to-validate-requests",
 "read_url_prefix": "${read-url-prefix}"
}
```

# Future Design Goals

## Extensibly limiting membership sets

Some service providers may wish to provide hub services to a limited set of
different users, with a provider-specific method of authenticating that a user
or address is within that set. In order to provide that functionality, our hub
implementation would need to be extensible enough to allow plugging in different
authentication models.

## A `.storage` Namespace

<!--- Blockstack Core no longer does this, but Gaia nodes do.  I updated this
section to reflect that (jcn) -->

Gaia nodes can request data from other Gaia nodes, and can store data
to other Gaia nodes.  In effect, Gaia nodes can be "chained together"
in arbitrarily complex ways.  This creates an opportunity to create
a decentralized storage marketplace.

### Example

For example, Alice can make her Gaia node public and program it to
store data to her Amazon S3 bucket and her Dropbox account.  Bob can then POST data to Alice's 
node, causing her node to replicate data to both providers.  Later, Charlie can
read Bob's data from Alice's node, causing Alice's node to fetch and serve back
the data from her cloud storage.  Neither Bob nor Charlie have to set up accounts on
Amazon S3 and Dropbox this way, since Alice's node serves as an intermediary
between them and the storage providers.

Since Alice is on the read/write path between Bob and Charlie and cloud storage,
she has the opportunity to make optimizations.  First, she can program her
Gaia node to synchronously write data to
local disk and asynchronously back it up to S3 and Dropbox.  This would speed up
Bob's writes, but at the cost of durability (i.e. Alice's node could crash
before replicating to the cloud).

In addition, Alice can program her Gaia node to service all reads from disk.  This
would speed up Charlie's reads, since he'll get the latest data without having
to hit back-end cloud storage providers.

### Service Description

Since Alice is providing a service to Bob and Charlie, she will want
compensation.  This can be achieved by having both of them send her money via
the underlying blockchain.

To do so, she would register her node's IP address in a
`.storage` namespace in Blockstack, and post her rates per gigabyte in her node's
profile and her payment address.  Once Bob and Charlie sent her payment, her
node would begin accepting reads and writes from them up to the capacity
purchased.  They would continue sending payments as long as Alice provides them
with service.

Other experienced Gaia node operators would register their nodes in `.storage`, and
compete for users by offerring better durability, availability, performance,
extra storage features, and so on.

# Notes on our deployed service

Our deployed service places some modest limitations on file uploads and rate limits.
Currently, the service will only allow up to 20 write requests per second and a maximum
file size of 5MB. However, these limitations are only for our service, if you deploy
your own Gaia hub, these limitations are not necessary.

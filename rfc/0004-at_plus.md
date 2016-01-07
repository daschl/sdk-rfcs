# Meta

 - RFC Name: AT_PLUS Consistency for N1QL Queries
 - RFC ID: 0004-at_plus
 - Start Date: 2015-12-16
 - Owner: Michael Nitschinger (@daschl)
 - Current Status: Draft

# Summary
This RFC specifies the user API and interaction with the Server for enhanced
RYOW query performance ("AT_PLUS" with Mutation Tokens).

# Motivation
The current approach to perform RYOW ("Read Your Own Writes") is to use the
REQUEST_PLUS scan consistency when executing a N1QL query. While this is
semantically correct, it takes the performance hit of having to pick the absolute
latest sequence number in the system and as a result potentially waiting for
a longer time until the indexer has caught up (especially under high mutation
  rates).

One way to mitigate these effects is to only wait until specific mutations have
been indexed instead of waiting for all of them up to the latest one. Also, the
likelihood of the indexer already having indexed the mutation is directly
proportional to the time delta between mutation and subsequent query.

Note that AT_PLUS is treated as a pure optimization over REQUEST_PLUS under
certain workloads and has no semantic difference to it in terms of correctness
inside the bounds specified through the mutation tokens.

# General Design
In the big picture of AT_PLUS for the developer, there are two components that
must be considered:

 1. The Mutation Token(s) must somehow be retrieved.
 2. The Token(s) need to be used at n1ql query time.

This RFC does not focus on 1. since it has been implemented already as part of
the enhanced durability requirements initiative in the 4.0 timeframe. This RFC
assumes that the Mutation Token is (at least) available as a return value from
a mutation operation, most likely exposed on the `Document` or equivalent.

Since it is important to this RFC, the properties of a Mutation Token are as
follows:

 1. A Mutation Token is opaque to the user and needs to be treated as such.
 2. Since it is opaque it is considered as immutable.

The internal structure of a Mutation Token is a triple which contains:

```
MutationToken {
    [required] vbucket_id = long
    [required] vbucket_uuid = long
    [required] sequence_number = long
}
```

The N1QL query execution for AT_PLUS consists of the following user level API:

```
consistent_with(ids: String...)
consistent_with(tokens: MutationToken...)
consistent_with(buckets: Bucket...)
```

For languages without overloading, the equivalents are:

```
consistent_with_ids(ids: String...)
consistent_with_tokens(tokens: MutationToken...)
consistent_with_buckets(buckets: Bucket...)
```

If more than one option is specified from the list above, an error must be raised
to indicate this user error (see errors).

The following describes the different options in detail and their user-facing
behavior:

 - `consistent_with_tokens`: 1 or more mutation tokens need to be passed in,
    and they will be used directly for the underlying `AT_PLUS` query. No other
    tokens are loaded and the user is 100% in charge of the lower query boundaries.
 - `consistent_with_ids`: The user passes in one or more document ids. The SDK
    transparently maps the String into a vbucket ID and grabs the tokens for all
    the ids passed in. Those tokens are used for the resulting n1ql query.
 - `consistent_with_buckets`: The user passes in on or more `Bucket` references.
    The SDK transparently grabs all (i.e. 1024 on linux) mutation tokens for the Bucket
    and uses it for the n1ql query. Note that this has quite significant overhead
    on the network, since 1024 tokens serialized take up many kilobytes per request.
    For this reason, if the user is concerned about low overhead the other 2 options
    should be preferred.

Note that snake or camel case is of course up to the best practices on the
target platform.

Setting `SCAN_CONSISTENCY` to anything but the default is considered a runtime
error (see errors). This means that `AT_PLUS`, while needed internally, is never
actually exposed to the user level. The user only specifies the consistentWith
boundaries and the SDK needs to figure out the rest (see implementation).

## Internal implementation
The N1QL Query API specifies the following API to be used for `AT_PLUS`:

```
{"scan_vector": {
    "<vbucket_id>": {
      "value": <sequence_number>,
      "guard": "<vbucket_uuid>"
    },
    "<vbucket_id>": {
      "value": <sequence_number>,
      "guard": "<vbucket_uuid>"
    }
  },
  "scan_consistency": "at_plus"
}
```

The `scan_vector` object contains one entry for each Mutation Token that needs
to be part of the index boundary. Explicitly note that the `guard` and
`vbucket_id` are strings while the `value` is a JSON number.

Once the query is assembled, the Mutation Tokens are marshaled into their JSON
representation and passed alongside all the other params to the N1QL query
engine.

How the mutation tokens are selected in the first place, depends on the type
of query issued by the user.

In its simplest form, if the user passes in the `MutationTokens` directly, they
are encoded and the query preparation is done. In the other two cases, it is slightly
more complicated:

- If Document ID's are passed in as Strings, the SDK needs to apply the same
  hashing logic as on the KV API to identify the correct vbucket ID for each
  document ID. Once the number of IDs is known, a central Map (see below) needs
  to be consulted which contains all the mutation tokens for every vbucket. The
  subset is extracted and then marshalled into the query.
- If the full bucket reference is passed in, all mutation tokens for this bucket
  (so up to 1024) need to be selected and marshalled into the query as shown
  above. This has the highest overhead over the network.

For both of these approaches to work, the SDK needs to maintain a `Token Map` for
each open Bucket which maps a vbucket to a MutationToken. To keep this map in sync,
after every mutation and if tokens are enabled, the map needs to be updated with
the latest token for the touched partitions. To rephrase this slightly: the SDK
needs to keep the state of all the mutations passed through it in the form of
Mutation Tokens per vbucket (partition).  

## Errors
The following user errors need to be covered by the SDKs:

| Error | Behavior |
| ----- | -------- |
| `SCAN_CONSISTENCY` specified in addition to `consistent_with` | IllegalArgumentException or equivalent thrown at runtime |
| More than one `consistent_with` variant used at the same time | IllegalArgumentException or equivalent thrown at runtime |

# Language Specifics

## Setting Scan Consistency With Document IDs

 - **Generic:** `consistent_with(ids: String...)`
 - **C:** ``
 - **Go:** ``
 - **Java:** `N1qlParams consistentWith(String... ids)`
 - **.NET:** `IQueryRequest ConsistentWith(params string[] ids)`
 - **NodeJS:** ``
 - **PHP:** ``
 - **Python:** ``
 - **Ruby:** ``

## Setting Scan Consistency With Mutation Tokens

- **Generic:** `consistent_with(tokens: MutationToken...)`
- **C:** ``
- **Go:** ``
- **Java:** `N1qlParams consistentWith(MutationToken... mutationTokens)`
- **.NET:** `IQueryRequest ConsistentWith(params MutationToken[] mutationTokens)`
- **NodeJS:** ``
- **PHP:** ``
- **Python:** ``
- **Ruby:** ``

## Setting Scan Consistency With Bucket Reference

- **Generic:** `consistent_with(buckets: Bucket...)`
- **C:** ``
- **Go:** ``
- **Java:** `N1qlParams consistentWith(Bucket... buckets)`
- **.NET:** `IQueryRequest ConsistentWith(params IBucket[] buckets)`
- **NodeJS:** ``
- **PHP:** ``
- **Python:** ``
- **Ruby:** ``

# Unresolved Questions

# Signoff

| Language | Representative | Date       |
| -------- | -------------- | ---------- |
| C        | | |
| Go       | | |
| Java     | | |
| .NET     | | |
| NodeJS   | | |
| PHP      | | |
| Python   | | |
| Ruby     | | |

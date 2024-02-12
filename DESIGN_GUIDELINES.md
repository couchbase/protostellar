# Design Guidelines

These protocol design guidelines describe the high level rules which are intended to be used throughout all aspects of the protocol.  The guidelines talk about naming conventions, inclusion/exclusion criteria, but do not speak to the specifics about how a particular method might work, or which features will be available in the protocol.

## General

- The design guidelines in this document MUST each justify their existence.

  - Specifying rules without justification can easily lead to rules being maintained against the betterment of the product, simply because nobody remembers why the rule exists.

- The design guidelines MUST NOT itself define the protocol, but instead be guidelines on how to define the protocol.

  - The goal of the guidelines is not to be a secondary description of the contents of the definition themselves, but rather a set of guidelines to help maintain consistency among many authors contributing to those definitions.

- New features MUST follow the structure and semantics of existing features except where updated design rules now apply.

  - Protostellar places a high value on the consistency across its surface area.  Consistency makes for an intuitive implementation which reduces the likelihood of bugs and unexpected behavior.

- The design rules MUST BE amended prior to implementing new structure or semantics which are not consistent with the existing rules.

  - This protocol is designed to be an open and distributed effort among the entire company and outlast the initial authorship.  This makes it challenging to maintain consistency across a wide range of contributors.  To this end, the protocol comes with a set of design guidelines that are intended to drive a level of consistency across this diverse group of contributors.  In order to maintain a level of consistency without preventing constant iteration and improvement of the protocol, the rules themselves are mutable through the same process as the protocol itself.

- Existing definitions MUST NOT be updated to adhere to new or amended design rules.

  - In spite of the desire to maintain consistency over the surface area of the protocol, cross-compatibility requirements make it difficult to change existing definitions.

- All commits to the protostellar definitions MUST contain a description explaining the reasoning behind the change.

  - Protostellar implementers and contributors exist throughout the company, it is valuable for anyone to be able to follow the changes to the protocol and understand their value.

- In the interest of maintaining consistency across the protocol, any time new definitions are introduced where questions arise from multiple possible options for implementation, the process that was used to make the decision SHOULD BE encoded into a rule in this document.
  - Similar to above, if decisions about the protocol are made but not well communicated, it's likely that a similar discussion would need to be had on multiple occasions.

## Field Types

- Types MUST NOT "wrap" another type purely for semantic reasons (e.g. CAS is a uint64 and must not be wrapped in a couchbase.Cas type).

  - This would introduce unnecessary bloat to data-types and garbage collection.

- Enumerations MUST BE preferred over arbitrary strings or numbers.

  - This would make the protocol more challenging to dissect and understand for implementers along with making validation and case handling unnecessary opaque.

- Enumerations MUST always start with a valid value.  There should not be any unset, default, etc... values present.  If specialized logic is intended to be used for default values, an optional field should be used instead.

  - This is in line with the expectation of not recreating functionality that already exists in protobufs.  Additionally, the inclusion of a specific enumeration value that represents a default leads to later compatibility issues, inclusion of unneeded data in serialized messages and restricts our ability to generically handle the concept of fields which are optional.

- Bitfield types MUST BE expressed as messages containing individual boolean fields.
  - Similar to above, this would otherwise make the protocol more opaque and challenging to validate and understand.

## Messages

- Message fields MUST BE optional unless executing the request would be invalid without the parameter.

  - This is a requirement related to our cross-version compatibility.  Fields being unnecessarily marked as required makes it more expensive to later remove or deprecate the field if necessary.

- Messages SHOULD BE compacted to the lowest possible depth possible except in cases where fields must be nested to achieve technical goals.
  - E.g. CAS would be specified as a uint64, not a couchbase.Cas.
  - E.g. total_rows in a query response would be QueryResponse.total_rows, not QueryResponse.metrics.total_rows.
  - E.g.  When using oneof, groups of fields may need to be grouped together in a separate message..
  - Nesting fields without a technical reason increases parsing and garbage-collection costs with little benefit to understandability.

## RPCs

- Bi-directional streaming MUST NOT be used.

  - As our protocol is designed to be request-response based, and servers are not permitted to send unsolicited data to clients, there is no need to have bi-directional streaming.

- Request-side streaming MUST NOT be used.
  - Protostellar is designed as a request-response protocol and thus has no current use-cases for supporting multiple requests being streamed for a single response.  This may change in the future to support certain cases of bulk operations.

## Naming

Generally speaking, the conventions described in this document should follow the naming conventions which were established by Google for their cloud products, along with the conventions that have been established by GRPC and Protobuf in their documentation.

### Namespacing

- Services which are specific to couchbase MUST exist within the couchbase namespace (ex: couchbase.data).

  - We should be clear to people that these definitions are specific to Couchbase.

- Services which provide administrative functionality (non-operational functionality) MUST place these operations into the couchbase.admin namespace (ex: couchbase.admin.data).

  - In the interest of separation of concerns, the administration of a service is considered a separate group of functionality from the service itself.  This will help clarify the distinction between these two sets of functionality.

- Namespaces MUST be split based on groups of functionality, rather than groups of implementation.  For example, ns_server provides Views, Routing and Management capabilities, but each of these individual blocks of functionality should be exposed as unique namespaced GRPC services.

  - Functionality available through Protostellar

- All namespaces MUST be suffixed with a version number, starting with v1.  For example, couchbase.data.v1.
  - In order to support future protocol updates that require breaking backwards interface compatibility, we should version the protocol based on this.

### Services

- Service names within a namespace MUST reflect the name of the namespace (with the exclusion of the couchbase portion) and only a single service MUST BE defined in each namespace.
  - e.g. couchbase.admin.bucket should contain a service called BucketAdminService.  Similarly, couchbase.data should contain a service called DataService.
  - This helps with maintaining a separation between the types that are specific to each service type.

### Methods

- Method names MUST be specified using CamelCase (with a leading capital).
  - This is a subjective consistency decision, we picked this as it matches most existing GRPC specifications.

### Messages

- Message name MUST be specified using CamelCase (with a leading capital).

  - This is a subjective consistency decision, we picked this as it matches most existing GRPC specifications.

- All field names MUST be specified in lower-case with underscores used as delimiter.  For example, routing_table.
  - This is a subjective consistency decision, we picked this as it matches most existing GRPC specifications.

## Versioning

- Versioning in this document is concerned with the versioning of the protocol itself, not the functionality or versioning of the server implementing it.  It is permissible for a server to implement or remove the implementation for a feature of the protocol based on its own rules.  For instance, a new protocol version may introduce a new optional field, but this field only works in a specific subset of server versions.  This enables new features to be introduced to the server without breaking clients unless the client utilizes the new feature (in which case, failure is expected if the server does not support it).  Additionally, this means that newer servers that have long-deprecated a feature can avoid implementing it, and older clients will correctly report that functionality as unsupported.

- Protocol versions are identifiable by a semver Major.Minor (no patch version).

- Protocol compatibility must not be broken within a major.

- Incrementing the major version is reserved for some far-future complete refactoring of the protocol.

## Entity Identification

Anytime an entity within the protocol must be identified a string identifier should be used.  Actors SHOULD generate these using UUIDv4.  Actors MUST ensure that these identifiers are globally unique (within reason, e.g. similar to the chances of a UUIDv4 identifier conflict).  These identifiers MUST NOT re-use other forms of ambiguous identification from the entity (e.g. you cannot use the DNS name of the system to uniquely identify the system, as this is mutable and a system may have multiple DNS names).

## Actor Identification

All Protostellar clients MUST generate a unique identifier for themselves.  This identifier MUST adhere to the [Entity Identification](https://docs.google.com/document/d/1_cCJ8ZTZv1RXHJTc3xrIHyQzHpKi3ifYaIQmt1me6Kg/edit#heading=h.y45sbaqjk9v) section above.  These identifiers should be used whenever a specific client needs to be identified by the protocol.

## Authentication

Authentication of Protostellar requests is handled through the use of an Authorization header or a TLS Client Certificate. In the case of an Authorization header, this may take the form of a service-specific Bearer token, or a general purpose username/password encoded of the standard Basic type.  We additionally support the use of a custom Authorization scheme "OnBehalfOf" to masquerade as another user.  This OnBehalfOf scheme works similarly to the Basic scheme with the addition of the On Behalf Of Users username being placed in front of the existing username/password.

Authorization: CB-OBO b2JvLXVzZXI6YWRtaW4tdXNlcjpwYXNzd29yZA==\
(decoded: obo-user:admin-user:password)

We utilize a custom scheme in the OBO case to ensure that intermediaries in the request chain which drop headers, or perform authorization of their own cannot successfully process the request without having an understanding of the necessary On-Behalf-Of semantics.  The use of the standard Authorization header ensures that intermediaries recognize the sensitive nature of the credentials encoded into the Authorization header.

Note that this OBO scheme does not enable the use-case where a server is client-certificate authenticated but also wishes to execute operations on behalf of another user.

Servers MUST perform Authorization header authentication even in the case that client-certificate authentication has already been performed.

## Versioning

We will use semantic versioning to describe Protostellar's version.  The patch version MUST always be omitted or assumed to be 0.

Backwards incompatible protobuf changes are not permitted in non-major versions.

Major version changes must be avoided, and if necessary must be planned far in advance as it triggers a client<->server cross-incompatibility.

## User Agents

Inevitably, bugs will be introduced into the implementation of the protocols in clients or servers.  In the interest of enabling the ability to provide version-specific workarounds in clients and servers should the need arise, each must identify itself with the other.

Clients MUST include a "User-Agent" header that indicates (at minimum) its software and version.  Servers MUST include a "Server" header that indicates (at minimum) its software and version.

## Error Handling

Error handling will utilize the standardized error code system that is built into GRPC (see GRPC Error Handling).  All errors that can occur within the Couchbase Server Architecture will be represented by a standardized error code, along with rich error context information (see GRPC Rich Error Handling) to provide more specific details on the underlying error that occurred.

Protobuf doc blocks should always clearly specify which errors are possible to be returned from each call, and the meaning of that error.  Additionally, at the service level, it should be documented which errors could be returned globally from any RPC call.

Error details should not contain information that is already known to the client ahead of time.  For instance, if you specify the bucket name in the request, it should not be included as part of any error response details.

- CANCELLED

  - Should be used in cases where an actor has explicitly canceled an operation, and this operation was successfully canceled.
  - Retries: No
  - Ambiguous: Yes.  The operation may or may not have succeeded.

- UNKNOWN

  - Should be used in cases where no information is available related to the cause of the error.  This can happen when an error code from another system is generated, and the intermediary does not understand it.
  - Retries: No
  - Ambiguous: Yes.  The operation may have partially completed.

- INVALID_ARGUMENT

  - Should be used in cases where the system cannot accept the arguments as provided.
  - Retries: No
  - Ambiguous: No.  The operation definitely failed.

- DEADLINE_EXCEEDED

  - Should be used in cases where a deadline was specified by an actor and the operation could not be completed before the deadline.
  - Retries: No.  The deadline should be extended rather than using retries.
  - Ambiguous: Yes.  The operation may or may not have succeeded.

- NOT_FOUND

  - Should be used in cases where an object was referenced which was not found.
  - Retries: No.
  - Ambiguous: No.

- ALREADY_EXISTS

  - Should be used in cases where an object was referenced with the expectation that it did not exist, but it was found to exist.
  - Retries: No
  - Ambiguous: No

- PERMISSION_DENIED

  - Indicates that the executing user does not have sufficient privileges.
  - Retries: No
  - Ambiguous: No.

- RESOURCE_EXHAUSTED

  - Indicates that some resource has been exhausted.  For instance user-quotas, disk-space, cluster capacity, etc...
  - Retries: No.
  - Ambiguous: No.

- FAILED_PRECONDITION

  - Indicates that an operation was rejected because the system is in the wrong state to be performing this operation.
  - Retries: No
  - Ambiguous: No

- ABORTED

  - Indicates that the operation was aborted due to some internal reason.
  - Retries: No
  - Ambiguous: No

- OUT_OF_RANGE

  - Indicates that a value was specified which is out of range of valid values.
  - Retries: No
  - Ambiguous: No

- UNIMPLEMENTED

  - Indicates that requested operation is not implemented.
  - Retries: No
  - Ambiguous: No

- INTERNAL

  - Indicates that some serious internal error has occurred.
  - Retries: No
  - Ambiguous: Yes

- UNAVAILABLE

  - Indicates transient unavailability of a service or functionality.
  - Retries: No, unless google.rpc.RetryInfo or couchbase.errors.v1.RedirectInfo attached.
  - Ambiguous: No

- DATA_LOSS

  - Indicates unrecoverable data-loss has occurred and the operation cannot succeed.
  - Retries: No
  - Ambiguous: No

- UNAUTHENTICATED
  - Indicates that the users authentication credentials were invalid or not provided.
  - Retries: No
  - Ambiguous: No

<https://grpc.github.io/grpc/core/md_doc_statuscodes.html>

## Backpressure

Backpressure behavior MUST NOT be implemented directly into the protocol, but instead the built-in backpressure and throttling capabilities that are integrated into HTTP2 and GRPC should be relied upon.

## Behavior Exclusivity

The protocol MUST NOT expose multiple mechanisms which lead to identical behavior.

For instance, the protocol did not initially provide a CreateIndex method for query indexes, as that behavior was already exposed through the execution of a CREATE INDEX statement (this was later amended as CreateIndex now provides consistency guarantees through additional logic provided beyond a simple query).  In opposition to this, something like WatchIndexes is something that should be exposed through the protocol since it is non-trivial coordination of other operations.  This includes the versioning cases, where one may be inclined to add methods such as `GetV2` where modifying `Get`'s definition would be possible (this does not preclude adding a `GetV2` if the behavior is fundamentally different).

## Session Stickiness

Session stickiness will explicitly not be required by Protostellar directly, although intermediate load balancer's are free to provide stickiness if the deployment environment could benefit from it.  It is thus inherently required that all RPCs can be executed independently, or that services implement background coordination with other instances such that requests are not required to be explicitly targeted.

## Retries

Retries should be executed at the lowest level possible.  For instance, except in the case of load-shedding, all servers should automatically retry requests which have failed for server-caused transient reasons.  In the case of a proxy service, the proxy service should retry these same requests that it has received in order to avoid unnecessary network traffic when possible.

## Consistency

In the interest of maintaining a useful developer experience, the protocol will enforce internal consistency when it comes to the management of resources.  For instance, creation of a bucket and then immediate use of that bucket MUST yield a successful operation, and having the second operation return NOT_FOUND would be a violation of this consistency.

Note that this requirement does not extend to requiring that the resource be available for use, simply that the service is aware of its existence.  For instance, creating a bucket and then attempting to write a document to it and receiving a form of temporary failure is acceptable.

## Routing

All exposed methods MUST enable full routing of the operation with the request data from the operation.  For instance, in the case of a Get operation, the Get request data must contain information to route the request to the appropriate cluster/bucket/scope/collection/key.  Note that this does not preclude the ability to include optional routing information through GRPC meta-data which optimizes or restricts the location of execution of an operation (for example, the Executor-ID meta-data mentioned in the Session Stickness section below).

The protocol MUST NOT include routing information which the server is responsible for resolving.  For instance, vbucket numbers must not be specified in the protocol as the server is ultimately responsible for the mapping of vbuckets.

The protocol will provide a mechanism to enable an operation being executed to indicate to the client that it must send the request to an alternative location.  This will be via the UNAVAILABLE error code, with additional Rich Error Context that indicates the location that the request must ultimately be dispatched to.  This redirection MUST be implemented as an "All Or Nothing" mechanism (ie: all method calls for the same resource must return a redirect, otherwise clients could easily become confused about whether the old or new location is the ultimately correct location).

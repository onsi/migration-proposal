# The Versioning Problem

Now that we've centralized access to Diego's database via the BBS server we're primed to attack the problem of data versioning & migration and API versioning (collectively referred to as "The Versioning Problem".

The goal of this proposal is to clearly define and ruthlessly simplify the problem statement as much as possible.  I'm trying to carve out a truly minimum viable approach to The Versioning Problem that serves as a launching pad for us to build the necessary operations experience and test infrastructure to attack more complex approaches (as driven out by, e.g. performance/uptime needs).

This MVP approach is relentlessly focused on getting Diego into production environments with reasonable guarantees around uptime between subsequent deploys.  Once in production we can iterate based on real-life feedback.  I'm also going to assume a post-receptor world in which all communication occurs directly with the BBS server.

## The Problem Statement

### What is an adequate solution to The Versioning Problem?

Our goal is to release a version of Diego (version 1.0.0) such that:

1. upgrades to subsequent versions can be accomplished with **minimal downtime** (see below)
2. we can iterate on Diego's features, add new functionality, and refactor Diego without excessive fear or concern of breaking #1
3. we don't lock ourselves into a versioning strategy that will be difficult to change

Let's agree on **minimal downtime**.  We propose that during a rolling deploy:

- applications should remain routable and continue to emit metrics and logs throughout the rolling deploy.
- Diego's evacuation protocol should be carried out succesfully and application instances should migrate from Cell to Cell, as Cells roll, without incident.
- small (~5 minute) windows of API downtime (both read & write) are acceptable during a deploy so long as:
    + an identifiable status code/message is relayed to the user
    + the first two conditions hold

Diego should maintain these guarantees across a single major version boundary.  i.e. Diego `vM1.m1.p1` can be upgraded to Diego `vM2.m2.p2` with **minimal downtime** if `0 <= M2 - M1 <= 1`

### What needs versioning?

There are two primary aspects of Diego that require versioning.

#### 1. The Database

We've made a distinction between versioning the *encoding* (e.g. JSON, protobuf, encrypted protobuf) of the data in the database and versioning the *schema* of the data in the database.  The *schema* is far ranging and entails both the data *fields* associated with individual models *and* the *layout* in the database (e.g. etcd key-paths).

My understanding is that the work we've done with the tasks allows us to include an encoding and version on each record in the database.  This is important primarily in the context of a failed migration.

If we *could* assume that migrations never fail we would not need to worry about the encoding/version for individual records.  We could have a single version that applies, globally, to the entire database.  However, since a migration *can* fail mid-flight.  We cannot rely on a single global version and must include encoding and version information in the individual models.  That way we can pick up a half-baked migration from where it left off.

The presence of a global DB version, however, does allow us to get concrete about how we version and migrate the database.  We'll cover this in more detail [below](https://github.com/onsi/migration-proposal#the-bbs-migration-mechanism).  For now, we argue that in light of our understanding of **minimal downtime** it is OK for Diego to experience complete API downtime during a migration of the database version.  We should characterize the duration of this downtime as a function of data size and and keep tabs on it as we introduce new migrations.  See the discussion in the [Keeping Migration Times Short](#keeping-migration-times-short) section below.

#### 2. The API

Various bits of Diego talk to other bits of Diego.  As the cluster rolls, these bits end up in different versions - and yet they must still be able to communicate.  In particular, we have the following communication paths:

- Internal
    - `Rep => BBS`
    - `BBS => Rep` (`StopActualLRPInstance`)
    - `BBS => Auctioneer => Rep`
- External
    - `CCBridge => BBS`
    - `Route-Emitter => BBS`

Versioning the API entails ensuring that these communication flows can happen even when the cluster is in a mixed state.  Since most communication is of the form X => BBSwe are in pretty good shape and can focus on reasoning about versioning access to the BBS.

There are two exceptions which we will deal with here:

##### `BBS => Rep` (`StopActualLRPInstance`)

This is a profoundly simple endpoint and it is unlikely to give us too much trouble in the future.  At a minimum we should version the endpoint on the Rep (i.e. `/v1/stop`).  If we ever choose to change this it is simply a matter of maintaining (then deprecating) the `/v1/stop` endpoint and teaching the client to try the `/v2/stop` endpoint and fall back if it is unavailable.

##### `BBS => Auctioneer => Rep`

Here too the surface area is small.  While it is unlikely for the *actions* to change (e.g. BBS=>`Auctioneer:RequestLRPAuctions` and `Auctioneer=>Rep:GET /state` and `Auctioneer=>Rep:POST /work`) it is quite likely that the *nouns* may change.  I.e. the data transferred back and forth by these end-points may need to be versioned.

This is something that could be handled somewhat transparently in the auction package. One straightforward thing that we might consider doing up front is to limit auction communication to focus purely on *resources* and *identifiers*.  For example, the `LRP`, `Task`, and `Resources` structs in the [Auction types package](https://github.com/cloudfoundry-incubator/auction/blob/master/auctiontypes/types.go#L126-L147).  This allows us to only worry about versioning data specific to the auction.

We don't currently do this, and it's a problem that we should address before we ship.  In particular, we schlep the entire `DesiredLRP` and `Task` payload through the auctioneer.  Not only is this inefficient, it is also unnecessary and complicates the versioning problem.  Far better would be for the `BBS` to request auctions that are purely comprised of identifiers and resources (again, `LRP` and `Task` in the [Auction types package](https://github.com/cloudfoundry-incubator/auction/blob/master/auctiontypes/types.go#L126-L147) are exactly what I'm talking about) and have the Rep request the relevant `DesiredLRP` and `Task` at container creation time.  This allows us to consolidate all versioning of `DesiredLRP` and `Task` in the `X => BBS` direction.

## Three Simplifying Assumptions

In the spirit of striving for an MVP solution to The Versioning Problem, we now propose three simplifying assumptions.

### Single Master-Elected BBS Server

The introduction of the BBS Server has led to a substantial simplification of the Versioning problem.  Now, instead of worrying about versioning access to the data format and schema in the database across different components, we have centralized the problem and can reason about how the BBS Server manages data versioning in the database.

However, the presence of multiple BBS Servers continues to be a source of complexity that is impeding quick and steady progress towards an MVP solution to The Versioning Problem.

We argue that a simpler starting point would be to have a *single* master-elected BBS Server handle BBS requests.  We discuss the migration flow and state machine for the master-elected BBS server in detail in [The BBS Migration Mechanism](#the-bbs-migration-mechanism) section below.

In this section we present a set of pros and cons with having a single-elected BBS server:

- Pros
    + [Substantially simpler migration mechanism](#the-bbs-migration-mechanism) accomplished with a single BOSH deploy
    + Migration transactionality guaranteed by having a single master node manage the migration
    + No complex distributed concensus mechanism necessary (with multiple readers it is necessary to keep the BBS cluster in agreement about what version to read/write, and whether a migration is occurring)
    + Uses familiar and proven patterns (master-elected leader)
    + Aligns with industry standards - both Borg and Mesos (arguably the most scalable mainstream schedulers out there) operate with single masters.
    + Single master is easier to reason about and performance tune.  e.g. can implement caching in a single master without worrying about cache-drift aross multiple readers.  Can even go so far as have a write-through cache.
    + Does not lock us in.  Free to iterate towards a multi-reader/multi-writer scenario later *if data deems it necessary*.  Avoids premature optimization and allows for data-driven optimization.
    + Simplest possible step to solve the MVP Versioning Problem
- Cons
    + Single bottleneck for data flow (though this is already the case with etcd as we *must* enable consistent reads, also - we should measure the implications of this, not simply assume it won't work)

### BBS-First Deploys

First off: I don't think this simplification is absolutely necessary.  However, I do believe the first few API bumps we write could abide by this principal.  As we learn from production environments (PWS, Blumix (eventually)) and our tests we can change tact.  The good news is that we can relax this assumption later and defer worrying about it till then.

#### Win #1: Simpler Clients

We have argued above that the primary API problem to solve is in the `X => BBS` direction.  Other communication paths have a small surface area and can be reasoned about separately - in particular, we've argued for improvements to the Aucitoneer communication path that substantially reduce the risk of us getting in trouble with incompatible Auction changes.

`X => BBS` communication entails a component `X` using a client that communicates with the `BBS`.  When upgrading from `vFoo` to `vBar` all the following states are possible:

| `X vFoo` | `X vBar`
---|----------|------------
`BBS VFoo` | Yes | Yes
`BBS VBar` | Yes | Yes

However, if we require the `BBS` upgrade to occur first then we only need to worry about:

| `X vFoo` | `X vBar`
---|-----------|----------
`BBS VFoo` | Yes | No
`BBS VBar` | Yes | Yes

This means that newer clients do not need to know how to communicate with older BBSes.  This could be a substantial simplification as it offloads the versioning burden completely onto the *BBS server*.  Clients, therefore, do not need to know which version of the BBS they are talking to.  They simply make requests and rely on the server to be able to satisfy those requests.

The server, on the other hand, must be able to respond to all requests within the supported version range.  Concretely, this means that the BBS server with version `vMs.ms.ps` must support requests from any client with version `vMc.mc.pc` where `0 <= Ms - Mc <= 1`.  These version identifiers could come in via an appropriate HTTP header, or via versioned HTTP paths (e.g. `/vM/foo` or `/vM.m/foo`).  By centralizing all the versioning code in the BBS server we can avoid having to reason about both the client and the server and we can build out substantial test suites to ensure this contract is maintained.

This is actually a pretty big deal.  When we add a new feature we may modify code in (say) the Rep to call a new method on a client with the latest version.  By not worrying about supporting older versions in the client we don't have to worry about supporting cases, in the Rep, where the desired codepath doesn't exist.  All we need to do is ensure the BBS can handle old *and* new requests.  This also allows us to unit test the various combinations of old/new versions in the BBS server alone.

#### Win #2: Early Feedback That a Deploy Isn't Going Well

If we require that the BBS roll first, and succefully transition from the current version to the next, we can be conservative around ensuring that all the forward migrations work before rolling the Cells.  This helps us ensure we are in a place where, for example, evacuation will not fail.  Moreover, if the BBS fails to come up, we can take more manual action to rollback (see below).

### No Rollbacks

As with the previous simplification, this is one we can undo later if we find we need to.  Rather than worry about supporting rolling back a Diego cluster up front I believe we can ship 1.0 without this support, but tack it on as a followup feature (perhaps after the performance work is done - we are free to prioritize as we wish).

Diego is built to be able to recreate the contents of ETCD in the case of catastrophic failure.  We want to end up in a place where we do not rely on this behavior - however, we can use this to our advantage and be aggressive about shipping 1.0 without having to fully solve the rollback scenario.  We could, instead, instruct operators to rollback to a previous version (ensure the cells go first), then nuke etcd and restart the BBS.

In reality, supporting rollbacks should not be particularly challenging, especially in the single-master BBS.  While I don't think we need this figured out for 1.0, here's a sketch of how a rollback could work:

1. We commit to implementing down-migrations for each up-migration.
2. Operators perform a manifest-only deploy that sets a flag on the BBS server to `--rollback-on-drain-to-version=ROLLBACK_TARGET`
3. Operators downgrade the cluster.  Cells roll first, followed by a deploy of the BBS cluster.  The first BBS to drain will perform the down migrations.  When the prior BBS version comes up it sees the version it expects and becomes master.

Note that, with this approach, we can implement rollbacks at a later point in time.  The earlier version of the BBS does not need to know how to participate in a rollback for the rollback to succeed.

> I am aware that SSH connectivity fails if we nuke the ETCD cache because the private keys stored int he DesiredLRPs get regenerated.  This is non-ideal but is OK, in my opinion, as it doesn't represent significant downtime.  Developers simply need to restart their app/kill an instance to regain SSH access.

## The BBS Migration Mechanism

Let's now dive into the details for the BBS Migration mechanism.  In this section we assume no support for rollbacks (the earlier section illustrates what rollback support could look like).

Here are 2 proposals.  A is more complex but paves the way for rollbacks a little more cleanly, feel free to jump down to B first.

### Proposal A - more complex, sets us up for rollbacks

We propose that the BBS have its data version, referred to as `BBSDataVersion` hard-coded into the binary (our proposed migration mechanism does not need any command line arguments).  `BBSDataVersion` should be a simple integer - semantic versioning applies at the top level of "Diego" and we don't think introducing semantic versioning down at the DB schema level is necessary.

The current state of the database is stored *in* the database under a `/version` key.  This key has a (JSON/Protobuf-encoded) value of the form:

```
type Version struct {
    CurrentVersion int
    TargetVersion int
}
```

When a BBS comes up it excercises the following state-machine control loop:

- Attempt to grab the lock
    + Upon failure, keep trying.
- Upon success, fetch `/version`
    + If the database version is more modern than the BBS version shut down. (see table below)
    + If the database version equals the BBS version, start serving requests. (see table below)
    + If the database version is less modern than the BBS version, start/continue the migration then serve requests. (see table below)
- During a migration a 503 response is returned to all requests.  Clients should be taught either to retry on a backoff or (perhaps more simply) forward the 503 along.
- Upon a successful migration, all requests should be handled.

Here's the comprehensive logic table for handling the various states of `Version`

`CurrentVersion` | `TargetVersion` | Action | Reason
----------------|-----------------|--------|-------
`nil` | `nil` | `END_MIGRATION` then `SERVE_REQUESTS` | **Expected**: a new BBS is talking to an empty database.  Mark the version as current and start handling requests.
`< BBSDataVersion` | `< BBSDataVersion` | `BEGIN_MIGRATION` then `END_MIGRATION` then `SERVE_REQUESTS` | **Expected**: a new BBS has been elected master for the first time and sees a database that needs migration.
`< BBSDataVersion` | `== BBSDataVersion` | `CONTINUE_MIGRATION` then `END_MIGRATION` then `SERVE_REQUESTS` | **Conceivable**: a new BBS failed mid-migration.  Let's try again.
`< BBSDataVersion` | `> BBSDataVersion` | `SHUT_DOWN` | **Inconceivable**: an operator took a cluster that had formerly failed to migrate and attempted to upgrade it.  Bad.
`== BBSDataVersion` | `< BBSDataVersion` | `SHUT_DOWN` | **Inconceivable***: for now.  In future versions this could be how a rollback is encoded.
`== BBSDataVersion` | `== BBSDataVersion` | `SERVE_REQUESTS` | **Expected**: database is up-to-date.  We just changed leaders.
`== BBSDataVersion` | `> BBSDataVersion` | `SHUT_DOWN` | **Conceivable**: a migration was taking place, but the master in charge of the migration died mid-migration and an older version of the BBS gained the lock.  It should let go and wait to be upgraded.
`> BBSDataVersion` | `< BBSDataVersion` | `SHUT_DOWN` | **Inconceivable**: this *could* happen in the future when we support rollbacks but it would represent a botched rollback.
`> BBSDataVersion` | `== BBSDataVersion` | `SHUT_DOWN` | **Inconceivable***: for now.  In future versions this could be how a rollback is encoded.
`> BBSDataVersion` | `> BBSDataVersion` | `SHUT_DOWN` | **Conceivable**: a migration took place, but the master in charge died.  An older version of the BBS gained the lock.  It should let go and wait to be upgraded.

And here's what the actions correspond to:

- `BEGIN_MIGRATION`: set `TargetVersion` to `BBSDataVersion` and begin migrating data.  During this time requests return 503.
- `CONTINUE_MIGRATION`: set `TargetVersion` to `BBSDataVersion` and continue migrating data (assumes migrations are idempotent).  During this time requests return 503.
- `END_MIGRATION`: set `CurrentVersion` and `TargetVersion` to `BBSDataVersion`.  
- `SERVE_REQUESTS`: start handling requests.
- `SHUT_DOWN`: release the lock.  Write nothing to the database.  Shut down.

### Proposal B - simpler, might need tweaks to support rollbacks

We propose that the BBS have its data version, referred to as `BBSDataVersion` hard-coded into the binary (our proposed migration mechanism does not need any command line arguments).  `BBSDataVersion` should be a simple integer - semantic versioning applies at the top level of "Diego" and we don't think introducing semantic versioning down at the DB schema level is necessary.

The current state of the database is stored *in* the database under a `/version` key.  This key has a (JSON/Protobuf-encoded) value of the form:

```
type Version struct {
    CurrentVersion int
    State VersionState // one of ACTIVE or MIGRATING
}
```

When a BBS comes up it excercises the following state-machine control loop:

- Attempt to grab the lock
    + Upon failure, keep trying.
- Upon success, fetch `/version`
    + If the database version is more modern than the BBS version shut down. (see table below)
    + If the database version equals the BBS version, start serving requests. (see table below)
    + If the database version is less modern than the BBS version, start/continue the migration then serve requests. (see table below)
- During a migration a 503 response is returned to all requests.  Clients should be taught either to retry on a backoff or (perhaps more simply) forward the 503 along.
- Upon a successful migration, all requests should be handled.

Here's the comprehensive logic table for handling the various states of `Version`

`CurrentVersion` | `State` | Action | Reason
----------------|-----------------|--------|-------
`nil` | `nil` | `END_MIGRATION` then `SERVE_REQUESTS` | **Expected**: a new BBS is talking to an empty database.  Mark the version as current and start handling requests.
`< BBSDataVersion` | `ACTIVE` | `BEGIN_MIGRATION` then `END_MIGRATION` then `SERVE_REQUESTS` | **Expected**: a new BBS has been elected master for the first time and sees a database that needs migration.
`< BBSDataVersion` | `MIGRATING` | `CONTINUE_MIGRATION` then `END_MIGRATION` then `SERVE_REQUESTS` | **Conceivable**: a new BBS failed mid-migration.  Let's try again.
`== BBSDataVersion` | `ACTIVE` | `SERVE_REQUESTS` | **Expected**: database is up-to-date.  We just changed leaders.
`== BBSDataVersion` | `MIGRATING` | `SHUT_DOWN` | **Conceivable**: a migration was taking place, but the master in charge of the migration died mid-migration and an older version of the BBS gained the lock.  It should let go and wait to be upgraded.
`> BBSDataVersion` | `ACTIVE` | `SHUT_DOWN` | **Conceivable**: a migration was taking place, but the master in charge of the migration died and an older version of the BBS gained the lock.  It should let go and wait to be upgraded.
`> BBSDataVersion` | `MIGRATING` | `SHUT_DOWN` | **Inconceivable**: an operator downgraded the cluster without nuking the database.  It had formerly failed to migrate.

And here's what the actions correspond to:

- `BEGIN_MIGRATION`: set `State` to `MIGRATING` and begin migrating data.  During this time requests return 503.
- `CONTINUE_MIGRATION`: set `State` to `MIGRATING` and continue migrating data (assumes migrations are idempotent).  During this time requests return 503.
- `END_MIGRATION`: set `CurrentVersion` to `BBSDataVersion` and `State` to `ACTIVE`.  
- `SERVE_REQUESTS`: start handling requests.
- `SHUT_DOWN`: release the lock.  Write nothing to the database.  Shut down.

### Handling failed migrations

If a migration fails mid-way we need to be able to idempotently rerun the migration.  This requires encoding the version of each individual model entry in the database.

## Versioning APIs and Clients

As outlined above, if we assume that the BBS always rolls first we can push all API versioning concerns into the BBS server.  Clients always call out to the API version hard-coded into the client.  The server is responsible for handling all older variants of the request.

We can accomplish this either via versioned endpoints (e.g. `POST /VAPI/foo`) or via an appropriate HTTP header.  Since the API version is external facing `VAPI` should conform to semantic versioning and should line up with Diego's semantic version.

At the level of the go client, we propose that a BBS server be tied to a version of the BBS client.  After a succesful rolling deploy all components should use the same BBS client version as the BBS server.  We propose, as outlined above, that a BBS server with version `vMs.ms.ps` must support requests from any client with version `vMc.mc.pc` where `0 <= Ms - Mc <= 1`.  This allows us to simplify the client by presenting a single interface to consumers.  The server can take care of handling and translating requests across different client versions.  The server may contain a whole host of versioned models (see the worked example below).

## A Worked Example

We've talked about splitting Desired LRPs into two (or more) objects. I'll explore a hypothetical approach for that as an example here.

#### The Version Change

This is not a major change so we can go from (e.g.) `v1.0` to `v1.1` when we implement this change.

#### Models in the BBS

We create two new messages in the BBS `MutableDesiredLRP` and `ImmutableDesiredLRP`, we then deprecated the old `DesiredLRP` message and the RPC stuff (RPC service protobuf is also imaginary here since we have a custom implementation for the time being):

```protobuf
// v1.0.0 -- deprecated
message DesiredLRP {
  option deprecated = true;

  // ...
}
```

```protobuf
message MutableDesiredLRP {
  optional string process_guid = 1;
  optional int32 instances = 2;
  optional bytes routes = 3 [(gogoproto.nullable) = true, (gogoproto.customtype) = "Routes"];
  optional string annotation = 4;
}

message ImmutableDesiredLRP {
  optional string process_guid = 1;
  // all the other fields...
}
```

#### Migrating Data

We write an `UpMigration` that:

1. Fetches the contents of the database into instances of `DesiredLRP`
2. Generates (in memory) `MutableDesiredLRP`/`ImmutableDesiredLRP` instances based on the `DesiredLRP`s
3. Writes the data

We can also (if we implement rollbacks) write a `DownMigration` that:

1. Fetches the contents of the database into instances of `MutableDesiredLRP`/`ImmutableDesiredLRP`
2. Generates (in memory) `DesiredLRP` with the contents of the `MutableDesiredLRP`/`ImmutableDesiredLRP` merged together again
3. Writes the data

#### The API

Since the API is also build of protobuf messages the change is similar. We now define new messages and deprecate the old ones:

```protobuf
// v1.0.0 -- deprecated
message CreateDesiredLRPRequest_1_0 {
  option deprecated = true;

  optional DesiredLRP desired_lrp = 1;
}

// v1.0.0 -- deprecated
message ListDesiredLRPsResponse_1_0 {
  option deprecated = true;

  optional Error error = 1;
  repeated DesiredLRP desired_lrps = 2;
}

message ListMutableDesiredLRPsResponse {
  optional Error error = 1;
  repeated MutableDesiredLRP mutable_desired_lrps = 2;
}

message ListImmutableDesiredLRPsResponse {
  optional Error error = 1;
  repeated ImmutableDesiredLRP immutable_desired_lrps = 2;
}

message CreateDesiredLRPRequest {
  optional MutableDesiredLRP mutable_desired_lrp = 1;
  optional ImmutableDesiredLRP immutable_desired_lrp = 2;
}

service DesiredLRPService {
  rpc ListMutableDesiredLRPs (ListMutableDesiredLRPsRequest) returns (ListMutableDesiredLRPsResponse);
  rpc ListImmutableDesiredLRPs (ListImmutableDesiredLRPsRequest) returns (ListImmutableDesiredLRPsResponse);
  rpc CreateDesiredLRP (CreateDesiredLRPsRequest) returns (CreateDesiredLRPResponse);

  // v1.0.0 -- deprecated
  rpc ListDesiredLRPs_1_0 (ListDesiredLRPsRequest_1_0) returns (ListDesiredLRPsResponse_1_0);
  rpc CreateDesiredLRP_1_0 (CreateDesiredLRPsRequest_1_0) returns (CreateDesiredLRPResponse_1_0);
}
```

##### `ListDesiredLRPs_1_0`

Fetches `MutableDesiredLRP`s and `ImmutableDesiredLRP`s merge them together into `DesiredLRP`s and sends `DesiredLRP` instances down the wire via the `ListDesiredLRPResponse_1_0`

##### `CreateDesiredLRP_1_0`

Expects a `CreateDesiredLRPRequest_1_0`, extracts an instance of `DesiredLRP` from it, then splits it into `MutableDesiredLRP`/`ImmutableDesiredLRP` and stores those.

No translation necessary for the newer methods since they can assume the data is already migrated.

#### Staying Sane

Not every story can end up with a version.  Rather, the team should be diligent about using release markers to delineate minor/major versions.  This, ideally, will collapse multiple features into minor version bumps and allow us to move at a reasonable velocity without exploding the matrix of backward compatibility.  Unit tests should be sufficient to ensure the older-version methods work (e.g. `ListDesiredLRPs_1_0`) though we will want an integration environment to ensure we can go from one major version to the next (see [Test Suites](#test-suites) below).

We should also be aggressive about bumping major versions (once a quarter?) and requiring folks to upgrade *through* major versions.

> One open question is how CI will work.  If we clump a bunch of stories together and indicate that v1.1 comes out when all the stories are finished how do we CI individual commits?

## Where we go in the post-1.0 era

After shipping a 1.0 release we are free to prioritize the following work in relation to the remaining tracks of work around performance and security.

#### Test suites

Ted's writeup[here](https://docs.google.com/document/d/1H2mD1ZAR9N4zVn_Y06FqpoYZwn0cf6i2o1EqorAD70I/edit#heading=h.4qopv1l933ib) has a detailed section on migration testing.  This is all good stuff that we should build out shortly after hitting 1.0.

#### Performance validated checks on whether or not we need more than 1 reader/writer

The biggest concern with a single BBS master is likeley going to be centered on performance.  We should validate these concerns with data and strive to optimize the singleton before deciding to embark on a substantially more complex distributed locking mechanism to ensure a cluster of BBSes can behave well during a migration.

#### Keeping Migration Times Short

Keeping this downtime window small could, and should, inform the sort of DB migration infrastructure we build.  For example, instead of a Rails-like migration system that layers on multiple migrations on the DB like so:

```
READ X1
X2 = migrate1To2(X1)
WRITE X2
DELETE X1
READ X2
X3 = migrate2To3(X2)
WRITE X3
DELETE X2
```

 we could build a migration system that operates completely in-memory:

```
READ X1
X2 = migrate1To2(X1)
X3 = migrate2To3(X2)
WRITE X3
DELETE X1
```

#### Maintaining Read Access during a Migration

Is this possible?  What might it look like?

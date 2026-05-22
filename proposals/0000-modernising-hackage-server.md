TODO:

* [ ] add details about the migration steps and requirements.
* [ ] add details if it's feasible to move only packagedb related parts to a relational database.
* [ ] something about the security

# Community Project Template \- Modernising Hackage Server

## Abstract

This proposal introduces the strategy for a complete and safe modernization and migration of the `hackage-server`, the central package repository for the Haskell ecosystem. Every developer, every CI pipeline, every minor operator depends on it. The current server is a decades-old legacy codebase that is suffering from architectural limitations, most notably the use of the unmaintained, in-memory data store `acid-state`, which prevents horizontal scaling and forces vertical scaling. Thus the Haskell Foundation has been [spending money on vertical scaling](https://github.com/haskell/hackage-server/issues/1467#issuecomment-4016939531) which is fundamentally more expensive than its horizontal counterpart. This document details these problems, explains why non-incremental change is necessary, and proposes a carefully conservative migration strategy, listing possible alternative solutions.

The proposed solution is a low-risk migration that involves building a new Postgres based backend system, incrementally behind the old one until the legacy system can be retired.

## Background

`hackage-server` is the central package repository for the Haskell ecosystem, essentially serving as the main library store for all Haskell developers. It is the single source for obtaining and managing packages, meaning that every developer, continuous integration (CI) pipeline, and operator relies on it for their daily work. `hackage-server` hosts all the packages, metadata, documentation, and revision history, and its functionality is fundamental for processes like building software, managing dependencies, and maintaining the broader Haskell community infrastructure.

The stability of the `hackage-server` is crucial because of its deep integration into every part of the Haskell development workflow. Its continuous availability ensures that projects can be built, dependencies can be resolved, documentation can be referenced, and new packages can be shared without interruption. If the service were to become unstable or fail, it would paralyze the entire ecosystem, affecting every person and automated system that depends on it.

The API contract for `hackage-server` is effectively frozen due to the extensive ecosystem of external tools that depend on it; any modifications would necessitate broad, complex coordination. Primary consumers that rely on this stable interface include `cabal`, `flora.pm`, `stackage-server`, and `hackage-server` itself when operating in mirror mode.

`acid-state` is a Haskell native persistent database library that *is expected to allow* storing and loading Haskell structures. It’s heavily used as a persistent layer in hackage-server. `acid-state` loads all data in memory allowing the use of arbitrary Haskell function for data querying and manipulation. In addition this library provides a *peculiar* framework for data migration.

## Problem Statement

Despite its importance, the current server is built on a decades-old legacy codebase with architectural limitations, which makes it costly and difficult to maintain. For instance, it uses an effectively-unmaintained, in-memory data store called `acid-state`, which forces all data to reside in RAM (currently 11GB to 16GB) on a single machine, preventing any meaningful horizontal scaling. This architecture leads to problems like server outages and forces the Haskell Foundation to spend money on expensive vertical scaling. To guarantee the long-term stability and performance required for critical infrastructure, both a complete modernization and migration are necessary.

To define concrete problems:

### 1\. A Persistence Layer That Cannot Scale

`hackage-server` uses `acid-state` for persistence: an in-memory data store that periodically checkpoints to disk. The entire application state must fit in RAM at all times. At time of writing, according to [the official server memory usage page](https://hackage.haskell.org/server-status/memory), this requires somewhere between 11GB and 16GB of RAM before any requests actually get served.

High memory usage in Hackage leads to problems with server outages (see  [hackage-server unresponsive · Issue \#1467](https://github.com/haskell/hackage-server/issues/1467#issuecomment-4016939531) for an example), which has historically been mitigated by vertical scaling.

What's worse, however, is that keeping the entire dataset in RAM means that it cannot be meaningfully scaled horizontally. The state gets updated in RAM and only occasionally saved on disk. Thus the state of the application must reside only on a single machine. The architecture does not allow for sharding or load-balancing, which are obvious wins for read-heavy workloads like hackage. Any attempts to horizontally scale `hackage-server` would have inconsistent reads, if the write replication hadn’t occurred yet, and `acid-state` gives no guarantees or notifications about when this happens.

### 2\. acid-state production readiness

*`acid-state` has entered a stage of strictly limited maintenance, with no ongoing development or performance objectives. This lack of progress prevents the library from achieving the operational maturity essential for critical infrastructure.*

Critical infrastructure demands a high degree of operational maturity, including robust observability, standardized backup and recovery protocols, and predictable failure resilience. It also requires support for high-availability patterns, performance-tuning capabilities, and seamless integration with modern database administration practices. Currently, `acid-state` fails to meet these standards; it has remained in basic maintenance mode: since 2022 it received only dependency updates. The library suffers from unresolved durability concerns, such as the mishandling of fsync errors [https://github.com/acid-state/acid-state/issues/175](https://github.com/acid-state/acid-state/issues/175), and lacks critical integration with modern observability frameworks like Prometheus or OpenTracing. Addressing these deficiencies through a simple audit is unfeasible, as true resolution would require extensive, ongoing maintenance and a total overhaul comparable in scope to this entire proposal. Furthermore, even with such efforts, `acid-state`'s fundamental limitations—specifically its lack of a distributed mode and the requirement to store the entire state in memory—would persist. In contrast, production-ready solutions like Postgres natively provide these essential features and benefit from active, professional maintenance.

The last but not the least problem with `acid-state`, is that `acid-state` couples the data model directly to the serialization format, any architectural refactoring becomes considerably more expensive.

### 3\. Accumulated Technical Debt

The codebase is further burdened by significant architectural technical debt beyond the primary data model and persistence issues. Although various aspects of the server could remain as they are, addressing the following liabilities during any modernization effort is highly advisable:

* Background tasks (e.g. haddock generation, email notifications, and running backups) are implemented via custom implementation of the cron daemon to schedule things like email notification and backups. Existing cron implementations solve the problem in all nuances and do not require a custom solution.
* Search is handled by a non-trivial implementation nearing 1000 LOC, with basic features and no tests, where better supported implementations are available.
* Arbitrary functionality is implemented via opaque `IO ()` callback hooks, making control flow hard to follow.
* Backup/restoring has a custom and complex implementation nearing 2500 LOC while better supported solutions are available.

## Prior Art and Related Efforts

Various components of `hackage-server` have seen independent reimplementations, most notably through projects like stackage-server, [flora.pm](http://flora.pm), and foliage. It is telling that despite being significantly more modern, none of these projects originated as a fork of the `hackage-server` codebase. We suspect this is due to a level of accumulated technical debt within `hackage-server` that makes forking less viable than starting anew.

### [Flora.pm](http://Flora.pm) and stackage-server

We could concentrate on the alternative implementation like [flora.pm](http://flora.pm) or stackage. But both those systems are built around existing hackage infrastructure and do not replace it. Moreover, some features are incompatible with current hackage.

The proposed approach concentrates on keeping full backwards compatibility and zero-downtime migration strategy that is much harder to implement around an already existing solution

### Foliage

Within the Cardano ecosystem, Foliage serves as a "static site generator" for index tarballs, creating cabal-compatible outputs. These repositories function as overlays for Hackage-server and can be integrated into `cabal.project` files using `source-repository-package` stanzas.

Although Foliage effectively addresses its specific use case, it is not intended to serve as a general-purpose package repository. While it represents an innovative exploration of the design space, the approach lacks extensive real-world testing; consequently, it is not considered a sufficiently battle-hardened foundation for the future of central infrastructure.

### Incremental migration by Tweag

Before proposing this project at Tweag we approached this problem using the iterative migration approach. During this work we have implemented a model and query layer that is compatible with current `hackage-server` architecture, however it was not possible to meaningfully integrate that work without a very large rewrite, see why “incremental refactoring is not feasible” section. The resolution was that we need wider change and resulted in this proposal. But the outcome was patches to `hackage-server` and model and query layer that can be reused in the current project.

## Technical Content

We propose a complete rewrite of `hackage-server`, into the following form. Although full rewrites are often hard to justify it is our opinion that this is the best approach forwards (see “Why Incremental Refactoring Is Not Feasible” and “Correctness Guarantees” for the specific details.)

The Hackage Server V2 project represents a *complete rewrite* of the existing infrastructure, utilizing contemporary Haskell libraries and development methodologies. This new version is architected into two primary segments:

* **The `hackage-api` library:** A new library featuring a Servant-based type definition that outlines the entire API surface. This component is designed to facilitate automatic bindings for modern downstream clients, as well as a machine-checkable specification for the actual implementation.
* **The `hackage-server` core:** A modernized implementation that replaces the legacy persistence layer with a PostgreSQL data store.

### Architecture

`hackage-server-v2` sits in front of `hackage-server` during the migration, proxying all requests to it if they are not yet implemented in v2, or if v2 experiences an exception.

Diagram 1\.

``` txt

              +-----------+                    +------+
              |    CAS    |<-------------------| CAS  |
              +-----------+    live migration  +------+
                    |                             |
+--------+   +------+------------+   +------------+---+
| fastly +---> hackage-server-v2 +---> hackage-server +
|  CDN   |   +---+----+----------+   +----------------+
+--------+       |    |
            +----+    |
            |         |
            v         |       +------------------+
       conformace     |       |                  |
          test        +------->   Postgres DB    |
                              |                  |
                              +------------------+

```

\~\~\~ — means that the component is temporary and it exists only for the duration of the migration.

Components (first new components, then already existing):

* **hackage-server-v2** — new implementation, consisting of the newly implemented endpoint, and programmable proxy that passed all the unknown requests to the existing `hackage-server`.
* **Postgres DB** — new component, a relational database where all current `hackage-server` state will be stored.
* **hackage-server** — (temporary component) current implementation that can remain unchanged on the course of all migrations.
* **fastly CDN** — a transparent caching layer that sits in front of the hackage server and serves static data (exists in the current architecture).
* **file storage**— storage where all the files, archives, documentation etc. are stored (exists in the current architecture).

#### Proxy

 A reverse proxy implemented in Haskell that for each route:

* Unmigrated routes proxy exclusively to `hackage-server`.
* Migrated read routes proxy to the new system, with fallback to `hackage-server` on error.
* Write routes duplicate to both systems during the dual-write period (Phase 3 below).

#### Hackage server v2

Our new implementation, as proposed above. All unknown requests are redirected to the existing hackage-server. All blob data is stored on content addressible storage, at this point we plan to use a solution that provide a filesystem like interface, we expect to use filesystem in the first step, but later it can be replaced by the FUSE that connects filesystem to S3 compatible storage.

To keep files up to date during the migration process we will keep live synchronisation using [lsync](https://github.com/lsyncd/lsyncd), that would allow to synchronize files, no matter how they were created, using upload, or by the background tasks.

Alternative solution is too use polling to fetch updates from the `hackage-server-v2`, the final solution will be chosen in cooperation with admin team based on security concerns and available options, as during the migration `hackage-server-v2` and `hackage-server` could be located in a different datacentres.

Fallback scenario that redirects requests from the `hackage-server-v2` to `hackage-server` when the file is not found will cover race conditions when file was not yet uploaded to `hackage-server-v2`. This fallback will be disabled once migration is complete.

#### Postgres DB

A database layer solution that keeps all the state. We chose Postgres for several reasons:

1. It’s a relational database and all the hackage state can be represented in a relation model, as was previously shown by stackage-server and flora PM. The relational model allows us to have a fast and efficient search model.
2. Postgres is a production grade database that is known as a good default for haskell applications and has multiple client libraries of the production quality.
3. Postgres contains multiple tools that cover observability, and reliability requirements and supports full text search out of the box. Many features of `hackage-server` v1 are functionality already provided directly by postgres; e.g. fulltext search, backups and pagination. These three features alone account for roughly 12% of the existing implementation of v1.
4. For the database migration we propose to use [sqitch](https://sqitch.org) library, that is a battle tested solution for migrations.

#### Physical locations

During the migration two boxes will be used:

1. Old physical box where `hackage-server` had been running, on this node we plan to keep hackage-server-v2, new storage and Postgres
2. Exising cloud solution for `hackage-server` is running, this box can be retired once migration is completed.

#### Why Servant?

We chose `servant` over competing web frameworks, since it provides a machine-consumable API in the form of a type. This allows us to be very explicit about what each route does, and the type-level machinery ensures that the implementation agrees with the specification.

### Why Incremental Refactoring Is Not Feasible

The natural response to a problematic codebase is incremental improvement: introduce abstractions, improve type safety, replace components one at a time. We have examined this path carefully and concluded that it is blocked.

The core data structure of `hackage-server` is approximately:

``` haskell
Map
  PackageName
  [ ( PackageIdentifier
    , [(Text,   (Time, UserId))]  -- cabal revisions
    , [(BlobId, (Time, UserId))]  -- tarball revisions`
    )
  ]
```

Large portions of the `hackage-server` codebase interact with this data structure directly, relying on ad-hoc filtering and manual transformations. The system lacks a dedicated query layer or abstraction, resulting in no clear distinction between the logical meaning of the data and its physical storage method. Consequently, whenever a component requires data, it must ingest and filter the entire database in-place as a pure value.

This severe architectural constraint forces any attempt at modifying the database backend to confront two challenging alternatives:

1. Maintain the existing logic by loading the complete database into RAM. This approach fails to reduce memory consumption and requires the new database to transmit its entire dataset every time the code executes.
2. Modify pure functions to allow IO access, enabling them to stream only the specific data they require.

The core issue is the IO boundary. Replacing `acid-state` with a real database requires IO, but the existing codebase assumes pure access to the full application state. Consequently, the first alternative leaves performance bottlenecks unaddressed, while the second requires very substantial engineering investment (see Appendix 1 for a thorough estimate of what needs to change, and how.)

Additional complexity arises from the fact that the two maintainers of `hackage-server` as listed in the cabal file haven't authored any commits since 2016 and 2013, respectively. The copyright field hasn't been updated since 2015\. There is no changelog. The original authors are no longer involved. The current maintainers did not write the system. There is little-to-no documentation of the architectural invariants, no record of why key decisions were made, and no single person who holds a complete mental model of how the pieces fit together.

As [Naur discusses](https://pages.cs.wisc.edu/~remzi/Naur.pdf), it is the "theory" of the software that is important, much more so than the artifact
itself. Without the theory, no significant changes can be made. The git history bears this out; with the exception of TUF and minor UI changes, no real features have been added to `hackage-server` since 2017\.

### Strangler Fig Migration

Rather than a flag-day replacement, we propose a strangler fig migration: a new backend is built incrementally behind the existing system, taking over traffic route by route until the old system can be retired. At no point is the ecosystem asked to trust an untested system with critical traffic.

Critically, this approach requires no changes to any downstream clients whatsoever, including mirror operators. Mirrors are pull-based consumers of Hackage's external interface. Because the migration is entirely encapsulated behind the same reverse proxy and the same endpoints, mirrors continue to function without any coordination, notification, or migration tooling. This is the only realistic migration strategy for infrastructure with an unbounded set of downstream consumers.

### Migration Sequence

For the migration we 5 distinct phases:

**Phase 1: Immutable content \-** Package tarballs are content-addressed and immutable. Serving these from the new system first is zero-risk, immediately proves the infrastructure, and represents a significant fraction of total request volume.

**Phase 2: Read-only structured data** \- Package metadata endpoints, the `.cabal` file endpoint, and other machine-readable structured data. During this phase, responses from both systems are compared and divergence is treated as a bug in the new implementation. To provide a better guarantee we plan to have dual reads, with verification of the results and switch to the new hackage only after we are confident that new replies are correct (see Correctness Guarantees).

**Phase 3: Dual writes** \- All state-mutating endpoints — uploads, revisions, maintainer changes, trustee actions, deprecations — write to both systems. The old system remains primary. This phase begins before the backfill.

**Phase 4: Backfill and sync** \- Historical data is backfilled into the new system's Postgres database. Because dual-writes began before the backfill, the two databases are guaranteed to converge: once the backfill reaches the point in time at which dual-writes started, the systems are in sync by construction and remain so. There is no write freeze, no coordinated cutover moment, and no race against live traffic.

**Phase 5: Cutover and retirement \-** Reads are switched to the new system. `hackage-server` is drained and retired.

This approach may look too superfluous, but it provides all required guarantees and is fully trackable by the community and allows to introduce changes earlier if required.

### Correctness Guarantees

To guarantee the new implementation remains indistinguishable to downstream consumers, we will conduct byte-for-byte consistency verifications on all machine-consumable outputs, such as JSON and tarballs, generated by the server. This process involves using the proxy to mirror live traffic to both versions simultaneously and comparing their responses. In the event of a discrepancy, the system will alert the developers and default to providing the output from the original version.

### Governance

We propose that the new system be owned by the Haskell Foundation from day one, with Tweag as the initial implementing contractor. This separates the question of who builds the system from the question of who stewards it, and ensures that the result is unambiguously community infrastructure rather than any single organization's property.

This structure also provides the long-term stability guarantee that the ecosystem requires: the system's continued operation does not depend on any single organization's continued involvement.

During the development all the code including draft patches and PR will be publicly open for external security reviews. The team will follow the best practices and do both manual and automatic reviews of all changes.

### Considered Alternatives

#### Scale Horizontally Anyway

We could run multiple copies of `hackage-server`, designate one as a write leader, and the others as readers. But `acid-state` keeps its state in RAM, so this would mean that readers wouldn’t immediately see new data. Furthermore, `acid-state` gives no hooks for subscribing to changes, so getting consistent reads would require high latency (waiting for readers to be restarted, and thus catch up), or custom code (to notify them).

Notably, this approach doesn’t lower the cost of adding hardware; machines still must be able to fit the entire dataset into RAM.

#### Acid-State Shim

One possibility involves implementing a shim layer over `acid-state`. While this would preserve current interfaces and semantics by backing the store with a distributable database like Postgres, it fails to address the memory bottleneck. Every horizontal node would still need to load the entire dataset into RAM, offering no significant cost advantage over the current vertical scaling approach.

Furthermore, the `acid-state` data model is inherently poorly suited for relational systems. This would force a choice between two suboptimal paths: storing serialized Haskell structures inefficiently or developing a complex, bespoke ORM for the legacy types.

Ultimately, this strategy is viewed as myopic; despite the considerable engineering effort required, it fails to resolve the underlying architectural issues.

#### Move packagedb to relational database, continue using `acid-state` for userdb

We could define a milestone around migrating only the package database. At that point, the packagedb implementation in hackage-server could be retired, reducing memory usage and addressing the most immediate operational concerns.

However, this partial migration would leave many of the underlying architectural issues unchanged. As discussed in *Why Incremental Refactoring is not Feasible*, the current codebase has deep coupling between persistence, I/O, and application logic. These boundaries are not cleanly separated, which makes incremental migration costly: significant refactoring effort is required even for narrowly scoped changes.

As a result, migrating only the packagedb would require much of the work associated with a broader redesign while delivering only a subset of the benefits. For this reason, we propose implementing the complete solution and migrating all package-related data to PostgreSQL rather than pursuing an intermediate state that would likely need to be revisited later.

## Timeline

We expect the entire project to be completed in 3 months after start, and include 2 engineers. See “Migration Sequence” for the intermediate steps of the technical work.

## Budget

We propose that this project may be completed by 2 engineers in 3 months. Excluding the time that should be spent on monitoring of the phase completions.

## Stakeholders

The main stakeholders are:

* Haskell Foundation — we expect that the costs on hardware and maintenance could be reduced significantly after the implementation of the proposal.
* Haskell community — is a secondary stakeholder, even if safety and stability of the haskell-server are important for the users, it’s currently cared for by other means (vertical scaling).
* The current infrastructure team for haskell-server, with sysadmin tasks handled by @gbaz. Although the initial phases of modernization may temporarily increase service reliability requirements as bundled components are transitioned into standalone systems, the long-term maintenance burden is expected to decrease. By utilizing industry-standard tools, operational knowledge can be distributed across the broader SRE community rather than remaining specialized within a small group.

## Success

The initiative will be deemed a success upon the effective migration and subsequent retirement of the legacy Hackage infrastructure. Success criteria include the new Hackage operating reliably without significant downtime while demonstrating improved resource efficiency. Furthermore, the updated architecture should facilitate horizontal scalability, allowing for load-balancing across standard commodity hardware.

# Appendices

* [Appendix 1: scope of refactoring](./0000-modernising-hackage-server/appendix1-scope-of-refactoring-work.md)
* [Appendix 2: existing routes](./0000-modernising-hackage-server/appendix2-existing-routes.md)
* [Appendix 3: scope of changing only packagedb](./0000-modernising-hackage-server/appendix3-scope-of-pkgdb.md)

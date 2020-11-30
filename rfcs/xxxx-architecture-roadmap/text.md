# cellxgene Architecture Roadmap


**Authors:** [Marcus Kinsella](mailto:mkinsella@chanzuckerberg.com)

**Approvers:**

## tl;dr 

cellxgene as a system has grown in complexity, and as part of that process, the original system design has become suboptimal. Moreover, the product roadmap
calls for more features and has new scaling requirements. We need to re-architect the system so we can better serve current and future user needs.

## Problem Statement

Architecture issues are discussed below in three parts:

- Overview of the current architecture
- Problems that exist today
- Future issues we'll encounter soon as the system grows

### Current Architecture

![Current Architecture](imgs/current_architecture.svg)

The cellxgene system architecture is divided into two big parts: the Portal and the Explorer.

Explorer responsibilities:
1. High-performance visualization of single-cell datasets. 
1. Calculation of gene differential expression among arbitrary subsets of cells.
1. Capturing and persisting annotation of subsets of cells for individual users.
1. Displaying metadata about the dataset: title, link to somebody's website, etc.
1. Enforcing simple authorization schemes: think public vs. private vs. bearer token.

Portal responsibilities:
1. Display, sorting, and filtering of datasets and collections of datasets.
1. Providing download links for datasets.
1. Directing users to Explorer visualizations of datasets.
1. Enforcing simple authorization schemes: think public vs. private vs. bearer token.
1. Ingesting and validating new datasets into existing or new collections.
1. Gathering metadata about collections and datasets.

Both parts implement their own auth, but they both use auth0 with the same client\_id. And there's not much authorization going on, both parts just parse the
`id\_token` to get a user id so they can figure out who owns what.

#### Explorer Details

Explorer components:
1. Frontend React app implementing the UI.
1. A Cloudfront distribution.
1. A load balancer receiving origin requests from Cloudfront.
1. A target group of EC2 instances handling requests from the load balancer.
1. An S3 bucket that holds datasets serialized in a TileDB format we call "cxg" as well as serialized user annotations.
1. An Aurora DB that maps (user, dataset) -> serialized user annotations.

The load balancer and target group and deployed via Elastic Beanstalk.

The FE makes three kinds of requests:
1. Data fetching. The FE needs things like embedding coordinates, gene expression values, and cell annotations. It sends requests for these through the
   ALB to an instance that reads the cxg file(s) in the bucket, transcodes them to a bespoke flatbuffer format, and gives that back to the FE. Each instance
   implements its own in-memory application cache that caches pretty aggressively based on assumed immutability of the underlying cxg files.
1. Compute. The FE can send requests to calculate differentially expressed genes in subsets of cells. The instance that gets this request fetches data from the
   bucket, performs the compute, and returns the result.
1. Recording annotations. When a user is authenticated, the FE can send a PUT request to record annotations of some cells and associate that with the user. The
   instance that receives the requests serializes it in cxg format in the bucket and updates the DB.

A fair bit of responsiveness and a reasonable time-to-interactive relies on flatbuffer efficiency and caching, both in the target group instances and in the
CDN. Also the compute capability is pretty limited by what a single processor can do before the request times out.

The user is identified by the user id ("sub") from the decoded `id\_token` from auth0.

The Explorer doesn't hold any internal record of what datasets are available or any metadata about them. As part of deployment, it's pointed at on or more
"dataroots". When it gets a request for a dataset, it does a `s/{base_url}/{dataroot}/` on the url and sees if it finds a cxg there. The dataroot value is
currently just something like `s3://hosted-cellxgene-prod`.

#### Portal Details

Portal components:
1. Frontend Gatsby app implementing the UI.
1. A Cloudfront distribution.
1. API Gateway that receives API requests from Cloudfront and directs them to Lambda functions.
1. A Batch compute environment that runs upload/conversion/validation jobs.
1. An S3 bucket that stores files associated with datasets.
1. An Aurora DB that stores information about datasets, collections, and their associated files.

The Portal's most compute-intensive task is validating uploaded datasets against a schema and converting them into other formats. It puts those formats into its
bucket. It also creates a cxg and puts that into the Explorer's bucket. It knows the base url of the Explorer, so it knows the url where users can access the
dataset in the Explorer.

The Portal also does some authorization. It tracks whether collections are private or public as well as who owns them.

### Current Problems to Solve

- **Poor integration boundary between Portal and Explorer**

The Explorer doesn't really know much about the Portal. That's good, you want keep the coupling between the two big parts of the system loose. But it does need
to integrate _a little_, and there's currently only one place it can do so: in the metadata of the serialized cxg in the Explorer bucket.

This means the interface is essentially undefined. The Explorer has to look in the cxg metadata and decide if it knows how to handle any of it. It needs to know 
about the metadata schema, which is a Portal concern. Changes to how the Portal writes metadata means there must be changes to the Explorer, and thus the
coupling is actually very tight.

Moreoever, the cxg's are more or less read-only. But the Portal needs to communicate new information about the dataset after it writes the cxg. For example,
perhaps there should now be a new publication associated with the dataset. Or the dataset is now public. There is no way to do this without rewriting the entire
cxg.

- **Explorer caching is mostly broken**

Previously, many requests from the Explorer could be served out of the in-memory cache of the instances or from Cloudfront. And the caching implementation
could be pretty straightforward because the underlying data was immutable, so no invalidation logic was needed, and we could scale horizontally without
introducing much complexity.

With user annotations enabled, the underlying data is no longer immutable. So now most requests require a full round trip to the database because there's never
any way to know that a cached response is still valid. And reasoning about horizontal scaling is much harder, as we need to handle data consistency. This
probably leads to particularly serious performance degradation for clients outside North America, as few of the requests can be served out of nearby POPs
anymore.

User annotations is a key new feature for the Explorer, but the core value proposition has always been performance, and we should work hard to avoid trading any
of that away.

- **99th %ile load time for the Portal is like 20 seconds**

The page is ~500kb worth of requests, and it's just absurd. I think Cloudfront actually makes it worse. You get a better median because of CDN cache hits, but
then the lambdas get cold, and the tail is so slow. Chalice/Lambdas is probably not the framework for our access patterns.

### Future Issues

There are also several requirements that will appear soon that will stress the current architecture:

- **Flexible, asynchronous compute**

Currently the only compute offered by the Explorer backed is differential expression of two cell subsets. The call to perform the compute is synchronous and
must completed before the browser or any of the backend components time it out. This limits the size of the cell subsets and the complexity of the algorithm.

Soon, the backend will need to offer multiple algorithms: multiple approaches to differential expression, re-embedding of cell subsets, automated cell labeling,
etc. These will introduce several new architectural requirements:

1. The computations will be long running; they will not complete in a single API call.
1. The number of available computations and their definitions will change independently of the rest of our architecture.
1. Computations may need to gather data from multiple sources: the core TileDB arrays, user annotations, model definitions, etc.
1. Computations may need to persist results beyond just the client session.

- **Authorization**

Currently our authorization is relatively simple: some entities must have an owner, and for those entities the identity of the requestor is matched against the
owner.

Soon, we will need to have entities with owners that are nevertheless "public", and there will be revokable, long-lived bearer tokens that permit access to some
entities. Authorization rules should propagate between Portal and Explorer, so if a user makes a collection public in the Portal, the associated Explorer
entities should become public as well.

After that, we will likely need slighty more complex authorization schemes, things like managing group membership or listing all collections a user can access.

- **Configuration management**

Currently configuration is updated in two ways depending on the change: re-deploying the whole system or re-writing cxgs. Both of these are slow, and this
approach is going to become less viable as the system gets more complex. For example, suppose the number of cell subset size for differential expression
changes, that should not require a full redeploy of the Explorer.

- **Maintaining desktop version**

As the system gets more complex, the distance between the hosted version and desktop version is going to grow. We'll need to be careful about maintaining
interfaces and behaviors in the hosted system that can be reproduced on a single computer running the desktop app.

## Proposed Architecture

![Proposed Architecture](imgs/proposed_architecture.svg)

### Explorer Changes

#### Split User-specific, Mutable Routes

We can regain caching and scaling in the primary 

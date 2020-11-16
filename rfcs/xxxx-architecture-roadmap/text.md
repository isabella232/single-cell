# cellxgene Architecture Roadmap


**Authors:** [Marcus Kinsella](mailto:mkinsella@chanzuckerberg.com)

**Approvers:**

## tl;dr 

We need to decompose our system a little bit. Right now we have poorly defined integrations between different components

## Problem Statement


#### Current Architecture

![Current Architecture](imgs/current_architecture.svg)

The cellxgene system architecture is divided into two big parts: the Portal and the Explorer.

Explorer responsibilities:
1. High-performance visualization of single-cell datasets. 
1. Calculation of gene differential expression among arbitrary subsets of cells.
1. Capturing and persisting annotation of subsets of cells for individual users.
1. Displaying metadata about the dataset: title, link to somebody's website, etc.

Portal responsibilities:
1. Display, sorting, filtering of datasets and collections of datasets.
1. Providing download links for datasets.
1. Directing users to Explorer visualizations of datasets.
1. Enforcing simple authorization schemes, think public vs. private vs. bearer token.
1. Ingesting and validating new datasets into existing or new collections.
1. Gathering metadata about collections and datasets.

Both parts implement their own auth, but they both use auth0 with the same client\_id. And there's not much authorization going on, both parts just parse the
id\_token to get a user id so they can figure out who owns what.

#### Explorer Details

Explorer components:
1. Frontend React app implementing the UI.
1. A load balancer listening at api.cellxgene.cziscience.com/cellxgene/
1. A target group of EC2 instances handling requests from the load balancer.
1. An S3 bucket that holds datasets serialized in a TileDB format we call "cxg" as well as serialized user annotations.
1. An Aurora DB that maps (user, dataset) -> serialized user annotations.

The load balancer and target group and deployed via Elastic Beanstalk.

The FE makes three kinds of requests:
1. Data fetching. The FE needs things like coordinates of cells, expression values of genes, and annotations of cells. It sends requests for these through the
   ALB to an instance that reads the cxg file(s) in the bucket, transcodes them to a bespoke flatbuffer format, and gives that back to the FE. Each instance
   implements its own in-memory application cache that caches pretty aggressively based on assumed immutability of the underlying cxg files.
1. Compute. The FE can send requests to calculate differentially expressed genes in subsets of cells. The instance that gets this request fetches data from the
   bucket, performs the compute, and returns the result.
1. Recording annotations. When a user in authenticated, the FE can send a PUT request to record annotations of some cells and associate that with the user. The
   instance that receives the requests serializes it in cxg format in the bucket and updates the DB.

A fair bit of responsiveness and a reasonable time-to-interactive relies on flatbuffer efficiency and caching. Also the compute capability is pretty limited
by what a single processor can do before the request times out.

The user is identified by the user id ("sub") from the decoded id\_token from auth0.

The Explorer doesn't hold any internal record of what datasets are available or any metadata about them. As part of deployment, it's pointed at on or more
"dataroots". When it gets a request for a dataset, it does a `s/{base_url}/{dataroot}/` on the url and sees if it finds a cxg there. The dataroot value is
currently just something like `s3://hosted-cellxgene-prod`.

#### Portal Details

Portal components:
1. Frontend Gatsby app implementing the UI.
1. API Gateway that directs API requests to Lambda functions.
1. A Fargate deployment that runs upload/conversion/validation jobs.
1. An S3 bucket that stores files associated with datasets.
1. An Aurora DB that stores information about datasets, collections, and their associated files.

The Portal's most compute-intensive task is validating uploaded datasets against a schema and converting them into other formats. It puts those formats into its
bucket. It also creates a cxg and puts that into the Explorer's bucket. It knows the base url of the Explorer, so it knows the url where users can access the
dataset in the Explorer.

The Portal also does some authorization. It tracks whether collections are private or public as well as who owns them.

### Problems to Solve

- Poor integration boundary between Portal and Explorer

The Explorer doesn't really know much about the Portal. That's good, you want keep the coupling between the two big parts of the system loose. But it does need
to integrate _a little_, and there's currently only one place it can do so: in the metadata of the serialized cxg in the Explorer bucket.

This means the interface is essentially undefined. The Explorer has to look in the cxg metadata and decide if it knows how to handle any of it. It needs to know 
about the metadata schema, which is a Portal concern. Changes to how the Portal writes metadata means there needs to be changes to the Explorer, and thus the
coupling is actually very tight.

Moreoever, the cxg's are more or less read-only. But the Portal needs to communicate new information about the dataset after it writes the cxg. For example,
perhaps there should now be a new publication associated with the dataset. Or the dataset is now public. There is no way to do this without rewriting the entire
cxg.

- Explorer caching is mostly broken

Previously, many requests from the Explorer could be served out of the in-memory cache of the instances. And the caching implementation could be pretty
straightforward because the underlying data was immutable, so no invalidation logic was needed, and we could scale horizontally without introducing much
complexity.

With user annotations enabled, the underlying data is no longer immutable. So now most requests require a full round trip to the database because there's never
any way to know that a cached response is still valid. And reasoning about horizontal scaling is much harder, as we need to handle data consistency.

User annotations is a key new feature for the Explorer, but the core value proposition has always been performance, and we should work hard to avoid trading any
of that away.

* 99th %ile load time for the Portal is like 20 seconds.

The page is ~500kb worth of requests, and it's just absurd. I think Cloudfront actually makes it worse. You get a better median because of CDN cache hits, but
then that lambdas get cold, and the tail is so slow. Chalice is probably not the framework for our access patterns.

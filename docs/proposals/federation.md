<!-- BEGIN MUNGE: UNVERSIONED_WARNING -->


<!-- END MUNGE: UNVERSIONED_WARNING -->

# Kubernetes Cluster Federation   

## (a.k.a. "Ubernetes")

## Requirements Analysis and Product Proposal

## _by Quinton Hoole ([quinton@google.com](mailto:quinton@google.com))_  

_Initial revision: 2015-03-05_  
_Last updated: 2015-03-09_  
This doc: [tinyurl.com/ubernetesv2](http://tinyurl.com/ubernetesv2)  
Slides: [tinyurl.com/ubernetes-slides](http://tinyurl.com/ubernetes-slides)

## Introduction

Today, each Kubernetes cluster is a relatively self-contained unit,
which typically runs in a single "on-premise" data centre or single
availability zone of a cloud provider (Google's GCE, Amazon's AWS,
etc).

Several current and potential Kubernetes users and customers have
expressed a keen interest in tying together ("federating") multiple
clusters in some sensible way in order to enable the following kinds
of use cases (intentionally vague):

1. _"Preferentially run my workloads in my on-premise cluster(s), but
   automatically overflow to my cloud-hosted cluster(s) if I run out
   of on-premise capacity"_.
1. _"Most of my workloads should run in my preferred cloud-hosted
   cluster(s), but some are privacy-sensitive, and should be
   automatically diverted to run in my secure, on-premise
   cluster(s)"_.
1. _"I want to avoid vendor lock-in, so I want my workloads to run
   across multiple cloud providers all the time.  I change my set of
   such cloud providers, and my pricing contracts with them,
   periodically"_.
1. _"I want to be immune to any single data centre or cloud
   availability zone outage, so I want to spread my service across
   multiple such zones (and ideally even across multiple cloud
   providers)."_

The above use cases are by necessity left imprecisely defined.  The
rest of this document explores these use cases and their implications
in further detail, and compares a few alternative high level
approaches to addressing them.  The idea of cluster federation has
informally become known as_ "Ubernetes"_.

## Summary/TL;DR

TBD

## What exactly is a Kubernetes Cluster?

A central design concept in Kubernetes is that of a _cluster_. While
loosely speaking, a cluster can be thought of as running in a single
data center, or cloud provider availability zone, a more precise
definition is that each cluster provides:

1. a single Kubernetes API entry point, 
1. a consistent, cluster-wide resource naming scheme
1. a scheduling/container placement domain
1. a service network routing domain
1. (in future) an authentication and authorization model.  
1. ....

The above in turn imply the need for a relatively performant, reliable
and cheap network within each cluster.

There is also assumed to be some degree of failure correlation across
a cluster, i.e.  whole clusters are expected to fail, at least
occasionally (due to cluster-wide power and network failures, natural
disasters etc). Clusters are often relatively homogenous in that all
compute nodes are typically provided by a single cloud provider or
hardware vendor, and connected by a common, unified network fabric.
But these are not hard requirements of Kubernetes.

Other classes of Kubernetes deployments than the one sketched above
are technically feasible, but come with some challenges of their own,
and are not yet common or explicitly supported.

More specifically, having a Kubernetes cluster span multiple
well-connected availability zones within a single geographical region
(e.g. US North East, UK, Japan etc) is worthy of further
consideration, in particular because it potentially addresses
some of these requirements.

## What use cases require Cluster Federation?

Let's name a few concrete use cases to aid the discussion:

## 1.Capacity Overflow

_"I want to preferentially run my workloads in my on-premise cluster(s), but automatically "overflow" to my cloud-hosted cluster(s) when I run out of on-premise capacity."_

This idea is known in some circles as "[cloudbursting](http://searchcloudcomputing.techtarget.com/definition/cloud-bursting)".

**Clarifying questions:** What is the unit of overflow?  Individual
  pods? Probably not always.  Replication controllers and their
  associated sets of pods?  Groups of replication controllers
  (a.k.a. distributed applications)?  How are persistent disks
  overflowed?  Can the "overflowed" pods communicate with their
  brethren and sistren pods and services in the other cluster(s)?
  Presumably yes, at higher cost and latency, provided that they use
  external service discovery. Is "overflow" enabled only when creating
  new workloads/replication controllers, or are existing workloads
  dynamically migrated between clusters based on fluctuating available
  capacity?  If so, what is the desired behaviour, and how is it
  achieved?  How, if at all, does this relate to quota enforcement
  (e.g. if we run out of on-premise capacity, can all or only some
  quotas transfer to other, potentially more expensive off-premise
  capacity?)

It seems that most of this boils down to:

1. **location affinity** (pods relative to each other, and to other
   stateful services like persistent storage - how is this expressed
   and enforced?)
1. **cross-cluster scheduling** (given location affinity constraints
   and other scheduling policy, which resources are assigned to which
   clusters, and by what?)
1. **cross-cluster service discovery** (how do pods in one cluster
   discover and communicate with pods in another cluster?)
1. **cross-cluster migration** (how do compute and storage resources,
   and the distributed applications to which they belong, move from
   one cluster to another)

## 2. Sensitive Workloads

_"I want most of my workloads to run in my preferred cloud-hosted
cluster(s), but some are privacy-sensitive, and should be
automatically diverted to run in my secure, on-premise cluster(s). The
list of privacy-sensitive workloads changes over time, and they're
subject to external auditing."_

**Clarifying questions:** What kinds of rules determine which
  workloads go where?  Is a static mapping from container (or more
  typically, replication controller) to cluster maintained and
  enforced?  If so, is it only enforced on startup, or are things
  migrated between clusters when the mappings change? This starts to
  look quite similar to "1. Capacity Overflow", and again seems to
  boil down to:

1. location affinity
1. cross-cluster scheduling
1. cross-cluster service discovery
1. cross-cluster migration
with the possible addition of:

+ cross-cluster monitoring and auditing (which is conveniently deemed
   to be outside the scope of this document, for the time being at
   least)

## 3. Vendor lock-in avoidance

_"My CTO wants us to avoid vendor lock-in, so she wants our workloads
to run across multiple cloud providers at all times.  She changes our
set of preferred cloud providers and pricing contracts with them
periodically, and doesn't want to have to communicate and manually
enforce these policy changes across the organization every time this
happens.  She wants it centrally and automatically enforced, monitored
and audited."_

**Clarifying questions:** Again, I think that this can potentially be
  reformulated as a Capacity Overflow problem - the fundamental
  principles seem to be the same or substantially similar to those
  above.

## 4. "Unavailability Zones"

_"I want to be immune to any single data centre or cloud availability
zone outage, so I want to spread my service across multiple such zones
(and ideally even across multiple cloud providers), and have my
service remain available even if one of the availability zones or
cloud providers "goes down"_.

It seems useful to split this into two sub use cases:

1. Multiple availability zones within a single cloud provider (across
   which feature sets like private networks, load balancing,
   persistent disks, data snapshots etc are typically consistent and
   explicitly designed to inter-operate).
1. Multiple cloud providers (typically with inconsistent feature sets
   and more limited interoperability).

The single cloud provider case might be easier to implement (although
the multi-cloud provider implementation should just work for a single
cloud provider).  Propose high-level design catering for both, with
initial implementation targeting single cloud provider only.

**Clarifying questions:**    
**How does global external service discovery work?** In the steady
  state, which external clients connect to which clusters?  GeoDNS or
  similar?  What is the tolerable failover latency if a cluster goes
  down?  Maybe something like (make up some numbers, notwithstanding
  some buggy DNS resolvers, TTL's, caches etc) ~3 minutes for ~90% of
  clients to re-issue DNS lookups and reconnect to a new cluster when
  their home cluster fails is good enough for most Kubernetes users
  (or at least way better than the status quo), given that these sorts
  of failure only happen a small number of times a year?

**How does dynamic load balancing across clusters work, if at all?**
  One simple starting point might be "it doesn't".  i.e. if a service
  in a cluster is deemed to be "up", it receives as much traffic as is
  generated "nearby" (even if it overloads).  If the service is deemed
  to "be down" in a given cluster, "all" nearby traffic is redirected
  to some other cluster within some number of seconds (failover could
  be automatic or manual).  Failover is essentially binary.  An
  improvement would be to detect when a service in a cluster reaches
  maximum serving capacity, and dynamically divert additional traffic
  to other clusters.  But how exactly does all of this work, and how
  much of it is provided by Kubernetes, as opposed to something else
  bolted on top (e.g. external monitoring and manipulation of GeoDNS)?

**How does this tie in with auto-scaling of services?** More
  specifically, if I run my service across _n_ clusters globally, and
  one (or more) of them fail, how do I ensure that the remaining _n-1_
  clusters have enough capacity to serve the additional, failed-over
  traffic?  Either:

1. I constantly over-provision all clusters by 1/n (potentially expensive), or
1. I "manually" update my replica count configurations in the
   remaining clusters by 1/n when the failure occurs, and Kubernetes
   takes care of the rest for me, or
1. Auto-scaling (not yet available) in the remaining clusters takes
   care of it for me automagically as the additional failed-over
   traffic arrives (with some latency).
1. I manually specify "additional resources to be provisioned" per
   remaining cluster, possibly proportional to both the remaining functioning resources
   and the unavailable resources in the failed cluster(s).
   (All the benefits of over-provisioning, without expensive idle resources.)

Doing nothing (i.e. forcing users to choose between 1 and 2 on their
own) is probably an OK starting point.  Kubernetes autoscaling can get
us to 3 at some later date.

Up to this point, this use case ("Unavailability Zones") seems materially different from all the others above.  It does not require dynamic cross-cluster service migration (we assume that the service is already running in more than one cluster when the failure occurs).  Nor does it necessarily involve cross-cluster service discovery or location affinity.  As a result, I propose that we address this use case somewhat independently of the others (although I strongly suspect that it will become substantially easier once we've solved the others).  
   
All of the above (regarding "Unavailibility Zones") refers primarily
to already-running user-facing services, and minimizing the impact on
end users of those services becoming unavailable in a given cluster.
What about the people and systems that deploy Kubernetes services
(devops etc)?  Should they be automatically shielded from the impact
of the cluster outage? i.e. have their new resource creation requests
automatically diverted to another cluster during the outage?  While
this specific requirement seems non-critical (manual fail-over seems
relatively non-arduous, ignoring the user-facing issues above), it
smells a lot like the first three use cases listed above ("Capacity
Overflow, Sensitive Services, Vendor lock-in..."), so if we address
those, we probably get this one free of charge.

## Core Challenges of Cluster Federation

As we saw above, a few common challenges fall out of most of the use
cases considered above, namely:

## Location Affinity

Can the pods comprising a single distributed application be
partitioned across more than one cluster? More generally, how far
apart, in network terms, can a given client and server within a
distributed application reasonably be?  A server need not necessarily
be a pod, but could instead be a persistent disk housing data, or some
other stateful network service.  What is tolerable is typically
application-dependent, primarily influenced by network bandwidth
consumption, latency requirements and cost sensitivity.

For simplicity, lets assume that all Kubernetes distributed
applications fall into one of three categories with respect to relative
location affinity:

1. **"Strictly Coupled"**: Those applications that strictly cannot be
   partitioned between clusters.  They simply fail if they are
   partitioned.  When scheduled, all pods _must_ be scheduled to the
   same cluster.  To move them, we need to shut the whole distributed
   application down (all pods) in one cluster, possibly move some
   data, and then bring the up all of the pods in another cluster.  To
   avoid downtime, we might bring up the replacement cluster and
   divert traffic there before turning down the original, but the
   principle is much the same.  In some cases moving the data might be
   prohibitively expensive or time-consuming, in which case these
   applications may be effectively _immovable_.
1. **"Strictly Decoupled"**: Those applications that can be
   indefinitely partitioned across more than one cluster, to no
   disadvantage.  An embarrassingly parallel YouTube porn detector,
   where each pod repeatedly dequeues a video URL from a remote work
   queue, downloads and chews on the video for a few hours, and
   arrives at a binary verdict, might be one such example. The pods
   derive no benefit from being close to each other, or anything else
   (other than the source of YouTube videos, which is assumed to be
   equally remote from all clusters in this example).  Each pod can be
   scheduled independently, in any cluster, and moved at any time.
1. **"Preferentially Coupled"**: Somewhere between Coupled and Decoupled.  These applications prefer to have all of their pods located in the same cluster (e.g. for failure correlation, network latency or bandwidth cost reasons), but can tolerate being partitioned for "short" periods of time (for example while migrating the application from one cluster to another). Most small to medium sized LAMP stacks with not-very-strict latency goals probably fall into this category (provided that they use sane service discovery and reconnect-on-fail, which they need to do anyway to run effectively, even in a single Kubernetes cluster).  

And then there's what I'll call _absolute_ location affinity.  Some
applications are required to run in bounded geographical or network
topology locations.  The reasons for this are typically
political/legislative (data privacy laws etc), or driven by network
proximity to consumers (or data providers) of the application ("most
of our users are in Western Europe, U.S. West Coast" etc).

**Proposal:** First tackle Strictly Decoupled applications (which can
  be trivially scheduled, partitioned or moved, one pod at a time).
  Then tackle Preferentially Coupled applications (which must be
  scheduled in totality in a single cluster, and can be moved, but
  ultimately in total, and necessarily within some bounded time).
  Leave strictly coupled applications to be manually moved between
  clusters as required for the foreseeable future.

## Cross-cluster service discovery

I propose having pods use standard discovery methods used by external clients of Kubernetes applications (i.e. DNS).  DNS might resolve to a public endpoint in the local or a remote cluster. Other than Strictly Coupled applications, software should be largely oblivious of which of the two occurs.   
_Aside:_ How do we avoid "tromboning" through an external VIP when DNS
resolves to a public IP on the local cluster?  Strictly speaking this
would be an optimization, and probably only matters to high bandwidth,
low latency communications.  We could potentially eliminate the
trombone with some kube-proxy magic if necessary. More detail to be
added here, but feel free to shoot down the basic DNS idea in the mean
time.

## Cross-cluster Scheduling

This is closely related to location affinity above, and also discussed
there.  The basic idea is that some controller, logically outside of
the basic Kubernetes control plane of the clusters in question, needs
to be able to:

1. Receive "global" resource creation requests.
1. Make policy-based decisions as to which cluster(s) should be used
   to fulfill each given resource request. In a simple case, the
   request is just redirected to one cluster.  In a more complex case,
   the request is "demultiplexed" into multiple sub-requests, each to
   a different cluster. Knowledge of the (albeit approximate)
   available capacity in each cluster will be required by the
   controller to sanely split the request.  Similarly, knowledge of
   the properties of the application (Location Affinity class --
   Strictly Coupled, Strictly Decoupled etc, privacy class etc) will
   be required.
1. Multiplex the responses from the individual clusters into an
   aggregate response.

## Cross-cluster Migration

Again this is closely related to location affinity discussed above,
and is in some sense an extension of Cross-cluster Scheduling. When
certain events occur, it becomes necessary or desirable for the
cluster federation system to proactively move distributed applications
(either in part or in whole) from one cluster to another. Examples of
such events include:

1. A low capacity event in a cluster (or a cluster failure).
1. A change of scheduling policy ("we no longer use cloud provider X").
1. A change of resource pricing ("cloud provider Y dropped their prices - lets migrate there").

Strictly Decoupled applications can be trivially moved, in part or in whole, one pod at a time, to one or more clusters.  
For Preferentially Decoupled applications, the federation system must first locate a single cluster with sufficient capacity to accommodate the entire application, then reserve that capacity, and incrementally move the application, one (or more) resources at a time, over to the new cluster, within some bounded time period (and possibly within a predefined "maintenance" window).  
Strictly Coupled applications (with the exception of those deemed
completely immovable) require the federation system to:

1. start up an entire replica application in the destination cluster
1. copy persistent data to the new application instance
1. switch traffic across
1. tear down the original application instance 

It is proposed that support for automated migration of Strictly Coupled applications be
deferred to a later date.

## Other Requirements

These are often left implicit by customers, but are worth calling out explicitly:

1. Software failure isolation between Kubernetes clusters should be
   retained as far as is practically possible.  The federation system
   should not materially increase the failure correlation across
   clusters.  For this reason the federation system should ideally be
   completely independent of the Kubernetes cluster control software,
   and look just like any other Kubernetes API client, with no special
   treatment.  If the federation system fails catastrophically, the
   underlying Kubernetes clusters should remain independently usable.
1. Unified monitoring, alerting and auditing across federated Kubernetes clusters.
1. Unified authentication, authorization and quota management across
   clusters (this is in direct conflict with failure isolation above,
   so there are some tough trade-offs to be made here).

## Proposed High-Level Architecture

TBD: All very hand-wavey still, but some initial thoughts to get the conversation going...

![image](federation-high-level-arch.png)

## Ubernetes API

This looks a lot like the existing Kubernetes API but is explicitly multi-cluster.  

+  Clusters become first class objects, which can be registered, listed, described, deregistered etc via the API.  
+  Compute resources can be explicitly requested in specific clusters, or automatically scheduled to the "best" cluster by Ubernetes (by a pluggable Policy Engine).  
+  There is a federated equivalent of a replication controller type, which is multicluster-aware, and delegates to cluster-specific replication controllers as required (e.g. a federated RC for n replicas might simply spawn multiple replication controllers in different clusters to do the hard work).  
+ These federated replication controllers (and in fact all the
   services comprising the Ubernetes Control Plane) have to run
   somewhere.  For high availability Ubernetes deployments, these
   services may run in a dedicated Kubernetes cluster, not physically
   co-located with any of the federated clusters.  But for simpler
   deployments, they may be run in one of the federated clusters (but
   when that cluster goes down, Ubernetes is down, obviously).

## Policy Engine and Migration/Replication Controllers

The Policy Engine decides which parts of each application go into each
cluster at any point in time, and stores this desired state in the
Desired Federation State store (an etcd or
similar). Migration/Replication Controllers reconcile this against the
desired states stored in the underlying Kubernetes clusters (by
watching both, and creating or updating the underlying Replication
Controllers and related Services accordingly).

## Authentication and Authorization

This should ideally be delegated to some external auth system, shared
by the underlying clusters, to avoid duplication and inconsistency.
Either that, or we end up with multilevel auth.  Local readonly
eventually consistent auth slaves in each cluster and in Ubernetes
could potentially cache auth, to mitigate an SPOF auth system.

## Proposed Next Steps

Identify concrete applications of each use case and configure a proof
of concept service that exercises the use case.  For example, cluster
failure tolerance seems popular, so set up an apache frontend with
replicas in each of three availability zones with either an Amazon Elastic
Load Balancer or Google Cloud Load Balancer pointing at them? What
does the zookeeper config look like for N=3 across 3 AZs -- and how
does each replica find the other replicas and how do clients find
their primary zookeeper replica? And now how do I do a shared, highly
available redis database?


<!-- BEGIN MUNGE: IS_VERSIONED -->
<!-- TAG IS_VERSIONED -->
<!-- END MUNGE: IS_VERSIONED -->


<!-- BEGIN MUNGE: GENERATED_ANALYTICS -->
[![Analytics](https://kubernetes-site.appspot.com/UA-36037335-10/GitHub/docs/proposals/federation.md?pixel)]()
<!-- END MUNGE: GENERATED_ANALYTICS -->

:tocdepth: 1

.. sectnum::

.. note::

   **This technote is a work-in-progress.**

Abstract
========

The First Look Analysis and Feedback Functionality working group has produced a set of requirements (`SITCOMTN-037`_) that include a Rapid Analysis capability (§3.1).  This document proposes a framework for executing this analysis and methods for enhancing consistency of algorithmic processing between Rapid Analysis, Alert Production, and other Science Pipelines uses.

.. _SITCOMTN-037: https://sitcomtn-037.lsst.io/

Framework
=========

Comparison of Rapid Analysis with Prompt Processing
---------------------------------------------------

The Prompt Processing (PP) framework (DMTN-219) has been designed to execute the Alert Production (AP) and potentially other PipelineTask pipelines at the US Data Facility (USDF).

There has been much discussion about using the Prompt Processing framework to execute Rapid Analysis (including some by the FAFF group itself, it appears from the report).
There are some key differences between the design of the former and the needs of the latter, however.

Prompt Processing requires a _sequence_ of multiple events, beginning well before an image is taken with ``nextVisit`` and continuing with ``image available for ingest`` for one or two snaps in a visit.
The ``nextVisit`` event is the key for preloading information needed by AP.

Rapid Analysis is intended to run after the receipt of a _single_ per-detector ``image in OODS`` event indicating that each image has been ingested into the Butler.
It then executes a script on that detector's image.
It should not be conditioned on a ``nextVisit`` event, as that event does not exist or is meaningless for many engineering and possibly Commissioning images that are not taken with normal observing scripts.

Triggering all Rapid Analysis on ``nextVisit``, even when no preload is necessary, will waste worker cores and memory that are sitting idle waiting for images.
This is essentially mandating two "strings" of processing for pipelines that may only need one (since they finish faster than the inter-exposure interval), or three for those that need two.

Prompt Processing requires elasticity of resources, as it is typically used for executing tasks that run for longer than the inter-visit (let alone the inter-exposure) time and can be variable in total runtime.
It is better to let processing (for which significant time has been invested) run long than to abort it and catch up later, but this requires bringing a spare worker online.
Enabling this elasticity adds some complexity and can lead to "cold starts", which are tolerable for Alert Production but not most potential Rapid Analysis uses.

Rapid Analysis can use fixed, pre-resourced deployments as the runtime of the analysis pipelines is expected to be short and well-characterized.

Comparison of Rapid Analysis and Auto-Ingest Service
----------------------------------------------------

On the other hand, there is an existing framework which will be used extensively within Data Management that seems closer to the key needs of Rapid Analysis.
This is the ``embargo_butler`` Auto-Ingest Service that currently runs at the USDF to ingest new images from the Summit into the Embargo Butler repo.
DM is already working at high priority to extend this service to ingest data into the OODS at the Summit, ingest data into the Butler repos associated with the Large File Annex (LFA) at the Summit and USDF, and ingest data into the USDF and UK and French Data Faciliities as it is replicated by Rucio during the Data Release Production.

In each of its deployments, this framework takes a single per-detector ``image available for ingest`` event (sometimes with associated metadata, in the Rucio case) and executes an IngestTask on it.
This can easily be repurposed to execute a more general script or Python function.

The Auto-Ingest Service uses fixed, pre-resourced deployments, as its workers are expected to complete their jobs rapidly.

Comparison of Auto-Ingest Service and RubinTV
---------------------------------------------

There are a few advantages of using the Auto-Ingest Service framework over the current `RubinTV framework`_.

.. _RubinTV framework: https://github.com/lsst-sitcom/rubintv_production

First, it is explicitly event-triggered, reducing latency and increasing efficiency over polling implementations.
Kafka will be used for reliable event delivery.
This is the *de facto* DM standard used within Sasquatch, PP, and the Rucio integration and soon to be used for SAL at the Summit.

Second, its event distributor is scalable, with any number able to run in parallel if one is insufficient.
Of course the back-end workers are (deployment-time) scalable, the same way that RubinTV's are.

Third, its event distribution mechanism can use a more-reliable Redis database rather than a shared filesystem.
This database can also be used to collect information that may need to be summarized across all the workers.
Avoiding a shared filesystem reduces the pressure on the Summit NFS filesystem, which is inherently a potential failure point.

While the auto-ingest task generally prefers to group requests into batches to minimize overhead, the batch size can (and has) been set to 1.

Conclusion
----------

Since DM is already productizing the Auto-Ingest Service framework for the LFA and Rucio uses, it will be minimal additional effort to ensure that it has a suitable interface for Rapid Analysis (as exemplified by the current RubinTV).
This framework is a better fit for Rapid Analysis than Prompt Processing, which will be allowed to develop on its own.

Attempting to modify Prompt Processing to satisfy Rapid Analysis's needs will take longer, leading to later delivery of the functionality, and it will require more effort, which is hard to find.


Pipelines
=========

RubinTV has needed customized versions of some tasks and pipelines in order to minimize latency and generate useful metrics, plots, and reports.
Many of these customizations can be similarly useful for Alert Production or for other pipelines running in the Prompt Processing framework.
We will arrange for existing tasks to add configuration to allow these customizations rather than requiring code modification, and we will ensure that configurations are properly inheritable so that changes made for scientific reasons are propagated to Rapid Analysis and changes made for analysis or performance reasons are propagated to Prompt Processing.

This synchronization will ideally result in common pipelines with small customizations rather than divergent pipelines with separate Tasks.


Summary
=======

Rapid Analysis will use an existing, maintained, reusable, scalable, reliable framework from Auto-Ingest, just not the Prompt Processing one.

Rapid Analysis will share Tasks and pipelines with Prompt Processing to the extent possible, with small customizations mostly through configuration rather than separate code.


.. Make in-text citations with: :cite:`bibkey`.
.. Uncomment to use citations
.. .. rubric:: References
.. 
.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :style: lsst_aa

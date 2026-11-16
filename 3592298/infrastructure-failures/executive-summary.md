# About 33 out of 100 failed pipelines encountered an infrastructure failure

Of the pipelines recorded as failed, how many encountered an infrastructure failure? About 33 out of 100: 245 of 738 failed pipelines (33.20%) had confirmed infrastructure evidence in at least one failed job attempt. The denominator here is only the failed pipelines—not every pipeline in the census. Infrastructure was involved, not necessarily the only cause; fixing it would not necessarily turn all 245 pipelines green.

Across all pipelines, how many encountered an infrastructure failure? About 20 out of 100: 344 of all 1,728 pipelines (19.91%). This is the same historical dataset and the same evidence rule, viewed with a broader denominator. It includes successful, canceled and running pipelines as well as failed ones; it is not a second data source or a conflicting estimate.

The difference is 99 affected pipelines that were not recorded failed: 91 successful, seven canceled and one running. A failed job attempt can later pass on retry, and successful pipelines can also contain permitted failures. We therefore do not claim that all 91 successful affected pipelines recovered through retries. Statuses are those recorded during the original collection, not their current states.

Both figures use the frozen creation window from 7 October 2026 at 08:34:26 UTC through 6 November 2026 at 08:34:26 UTC. Each affected pipeline is counted once, even if several jobs or retries failed. The census contains 95,740 job-attempt records, including 4,068 failed attempts; 772 confirmed infrastructure-failed attempts collapse to 344 distinct affected pipelines.

The other 493 failed pipelines have no confirmed infrastructure evidence in this audit. They are not automatically product bugs. Likewise, the other 1,384 pipelines in the all-pipeline view are not proven infrastructure-free. Mixed causes remain possible, and no exhaustive infrastructure-only, mixed or product-only causal split is established.

Confirmed evidence includes explicit shutdowns, memory-pressure or taint evictions, runner-backend TLS failures, and fatal registry, dependency, checkout or artifact transport failures. Ordinary test timeouts, unexplained container OOM, database availability symptoms, authentication errors and unknowns were not automatically counted. Categories overlap and must not be summed as extra affected pipelines.

These historical lower bounds support targeted investigation, not a guaranteed improvement in green pipelines or a current reliability estimate. Aggregate counts were recomputed from retained structured findings. Bulk raw logs were deleted under authorized retention; this is not a fresh full trace audit.

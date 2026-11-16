# At least 19.91% of pipelines encountered confirmed infrastructure failures

In a historical Canvas CI census, 344 of 1,728 distinct pipelines had at least one failed job attempt with confirmed infrastructure evidence. The frozen creation window is 7 October 2026 at 08:34:26 UTC through 6 November 2026 at 08:34:26 UTC. This includes all recorded pipeline statuses, with outcomes observed during collection rather than refreshed today.

Each affected pipeline is counted once, regardless of how many jobs or retries failed. Recovered and mixed-failure pipelines remain included. Infrastructure was involved, but was not necessarily the only cause: fixing it would not necessarily make every affected pipeline green. This is a historical lower bound, not a current failure rate.

The other 1,384 pipelines have no confirmed infrastructure evidence in the retained audit. That does not establish that they were infrastructure-free or that their failures were product bugs. Unknowns, ordinary test timeouts and unexplained out-of-memory symptoms were not automatically classified as infrastructure.

The downloadable evidence lists all 344 affected pipelines, their public links, recorded statuses, confirmed categories and failed job IDs. Aggregate counts and source hashes back the result. Category counts overlap and are not additive; retries never add pipeline units.

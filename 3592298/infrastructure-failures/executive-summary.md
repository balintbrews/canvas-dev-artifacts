# Infrastructure failures across 1,728 pipelines

**7 October–6 November 2026**

| Finding | Pipelines | Share of all 1,728 |
|---|---:|---:|
| Confirmed infrastructure failures | 344 | 19.91% |
| No confirmed infrastructure failures | 1,384 | 80.09% |

Each pipeline is counted once. Confirmed cases had at least one infrastructure-failed job attempt, including retries or later recovery.

**No confirmed infrastructure failure does not mean the pipeline passed.** Confirmed cases may also contain test failures.

## Confirmed failure types

| Type | Distinct pipelines |
|---|---:|
| Runner-node shutdowns | 298 |
| Container-registry access failures | 29 |
| Runner evictions (memory pressure or platform rules) | 8 |
| Dependency download failures | 8 |
| Runner backend TLS connection failures | 6 |
| Source checkout or artifact-transfer failures | 5 |

One pipeline can have multiple failure types. These counts overlap; do not add them.

[344-pipeline evidence · CSV](pipeline-evidence.csv) · [Methodology and source hashes · JSON](report-metadata.json)

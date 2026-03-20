You can keep AutoSys as the “front door” while incrementally moving all Unix box jobs into Airflow + Spring Batch, and still get full visibility and safe re‑runs from Airflow during the whole transition.[1][2][3]

## 1. Incremental Migration Pattern

- Phase 1 – Side‑by‑side:  
  - Keep existing AutoSys schedules as‑is but change the command of selected box jobs to “trigger an Airflow DAG run” (CLI or REST) instead of calling the legacy shell script directly.[4][5][1]
  - Inside the DAG, call your existing Spring Batch jar (or a containerized Spring Batch job) as one or more tasks, so Airflow orchestrates the steps and captures status/logs.[6]

- Phase 2 – Progressive expansion:  
  - Keep adding new jobs / refactors as Airflow DAGs, still triggered by AutoSys, until most critical flows are on Airflow.[2][3][7]
  - At this point, AutoSys is mostly a “scheduler shim”; Airflow is the real orchestrator and source of truth for job status.[1][2]

- Phase 3 – Cutover:  
  - For fully migrated workflows, move the cron schedule from AutoSys into Airflow (native `schedule`) and decommission the corresponding AutoSys jobs.[3][2]

This gives you a parallel‑run and canary model instead of a big bang.[7][3]

## 2. Triggering DAGs from AutoSys

- From AutoSys, configure the job command to call either:  
  - Airflow CLI on a jump host: `airflow dags trigger example_batch_workflow` (or with `-c` for params).[8][4]
  - A wrapper script that calls Airflow’s REST API `/api/v1/dags/{dag_id}/dagRuns` with a JSON body containing a run id or logical date.[5][4]

- Behavior in Airflow:  
  - Externally triggered DAG runs show up in the Airflow UI alongside scheduled runs; their status lifecycle is the same (queued, running, success, failed).[9][10]
  - You can pass identifiers from AutoSys (e.g., job name, Autosys run id) as `conf` payload; tasks can log them so you can correlate AutoSys and Airflow.[4][9]

This lets AutoSys own “when” and Airflow own “how” without losing traceability.[9][1]

## 3. Full Visibility of Status

- Airflow Web UI gives:  
  - DAG‑level views (tree, graph, Gantt, grid) with status for every task in each run.[10][9]
  - Per‑task logs stored in the configured backend (local, S3, GCS, etc.), accessible via the UI or API.[11][9]

- For external consumers (e.g., operations teams still living in AutoSys):  
  - Poll the Airflow REST API for DAG run / task instance status by `dag_id`, `dag_run_id`, or logical date, and surface a summarized status back into AutoSys (e.g., as an AutoSys job exit code or via an integrated view).[12][9]
  - Use Prometheus + Grafana on Airflow metrics (DAG run counts, failures, durations) for dashboards and alerting that complement the AutoSys view.[13][14][15]

So even while AutoSys is the scheduler of record, Airflow becomes the **visibility** and **observability** system for the new jobs.[2][1]

## 4. Re‑run Capabilities

- Re‑running from Airflow UI:  
  - To re‑run a whole DAG: clear the DAG run (or all failed tasks) for a given execution date/run id; Airflow sets task states back to `None` and re‑schedules them.[16][10]
  - To re‑run specific steps: click a failed task in the graph/tree view and choose **Clear**, optionally including upstream/downstream or only failed tasks.[10][16]

- Re‑running from AutoSys:  
  - You can model “rerun” as a separate AutoSys job that triggers the same DAG with a specific `run_id` or parameters (e.g., business date) in the `conf` payload, rather than editing old schedules.[4][9]
  - Since Airflow persists run history, you always have the audit trail of which AutoSys invocation re‑ran which data slice.[9][10]

This satisfies your requirement for controlled re‑runs without giving up on centralized scheduling while you migrate.[16][10]

## 5. Simple End‑State Diagram (Incremental View)

Conceptually, the long‑term incremental setup looks like:

```mermaid
flowchart LR
  AutoSys[AutoSys Scheduler] -->|Triggers| AirflowDAG[[Airflow DAG + Spring Batch]]

  subgraph AirflowCluster
    AirflowWeb[Airflow Web UI]
    AirflowSched[Airflow Scheduler]
  end

  AirflowDAG -->|Runs Steps| SpringBatchJob[Spring Batch Job(s)]
  SpringBatchJob -->|Reads/Writes| DB[(Database)]

  AirflowWeb -->|Status & Logs| OpsTeam[Ops / Support]

  Prometheus[Prometheus] -->|Scrapes Metrics| AirflowCluster
  Grafana[Grafana] -->|Dashboards| Prometheus
```

AutoSys remains the entrypoint, but Airflow is where you see status, logs, metrics, and perform re‑runs.[14][1][9]

***

To make this concrete, do you want a code sketch of a DAG that invokes a Spring Batch jar (or container) and passes through the AutoSys run id / date so you can fully trace a job across both systems?

Sources
[1] Apache Airflow Job - Broadcom Techdocs https://techdocs.broadcom.com/us/en/ca-enterprise-software/intelligent-automation/autosys-workload-automation/24-0-00/scheduling/ae-scheduling/apache-airflow-job.html
[2] Converting Legacy Schedulers to Airflow https://airflowsummit.org/sessions/2024/converting-legacy-schedulers-to-airflow/
[3] Replacing Legacy Schedulers with Apache Airflow - Video https://www.astronomer.io/events/webinars/modernizing-your-data-stack-replacing-legacy-schedulers-with-apache-airflow-video/
[4] How to Trigger Airflow Jobs in Python | by Lynn G. Kwong https://python.plainenglish.io/how-to-trigger-airflow-jobs-in-python-1ef15b08d70b
[5] Implementing Event-Driven DAG Execution in Apache Airflow https://blog.devgenius.io/implementing-event-driven-dag-execution-in-apache-airflow-80c305d5b3cf
[6] Batch vs Stream Processing in Apache Airflow Explained https://moldstud.com/articles/p-batch-vs-stream-processing-in-apache-airflow-explained
[7] Migrating Legacy Workflows with Airflow, Bamboo, and Bitbucket https://thinkcloudly.com/blog/migrating-legacy-workflows-with-airflow-bamboo-and-bitbucket/
[8] How to trigger DAG in Airflow everytime an external event state is ... https://stackoverflow.com/questions/74578403/how-to-trigger-dag-in-airflow-everytime-an-external-event-state-is-true-event-b
[9] Dag Run Status - Apache Airflow https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/dag-run.html
[10] DAG Run Status - Apache Airflow https://airflow.apache.org/docs/apache-airflow/2.3.3/dag-run.html
[11] Dags — Airflow 3.1.8 Documentation https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/dags.html
[12] Is there a way to externally check the status of each task in a DAG? https://stackoverflow.com/questions/69748839/is-there-a-way-to-externally-check-the-status-of-each-task-in-a-dag
[13] Need Airflow DAG monitoring tips : r/dataengineering - Reddit https://www.reddit.com/r/dataengineering/comments/1o2tgtq/need_airflow_dag_monitoring_tips/
[14] Monitoring Apache Airflow using Prometheus - Red Hat https://www.redhat.com/en/blog/monitoring-apache-airflow-using-prometheus
[15] Apache Airflow monitoring made easy | Grafana Labs https://grafana.com/integrations/apache-airflow/monitor/
[16] Rerun Airflow DAGs and tasks | Astronomer Documentation https://www.astronomer.io/docs/learn/rerunning-dags
[17] Externally Triggering DAGs - Good or Bad? : r/dataengineering https://www.reddit.com/r/dataengineering/comments/199w676/externally_triggering_dags_good_or_bad/
[18] Triggering a DAG from another DAG and wait for its completion https://github.com/apache/airflow/discussions/15049
[19] Airflow Datasets and Pub/Sub for Dynamic DAG Triggering - YouTube https://www.youtube.com/watch?v=4EU7M_E5pMw

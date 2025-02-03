# fuzzy-pancake

To create an autonomous worker in the context of the `crates/crates_io_worker/src/worker.rs` file from the `rust-lang/crates.io` repository, you can follow these steps:

1. **Define Worker Structure**: Ensure your worker structure includes necessary fields such as `connection_pool`, `context`, `job_registry`, `shutdown_when_queue_empty`, and `poll_interval`.

2. **Implement Worker Logic**:
   - Implement the `run` method to continuously process jobs by checking the queue and executing jobs until the queue is empty or the worker is shut down.
   - Implement the `run_next_job` method to handle the logic of fetching and executing the next job in the queue.

3. **Autonomous Worker Features**:
   - **Self-Scheduling**: The worker should be able to schedule itself to run at specific intervals without external triggers. This can be achieved using `tokio::time::sleep` for periodic polling.
   - **Error Handling and Retries**: Implement robust error handling and job retry mechanisms to ensure the worker can autonomously recover from failures.

4. **Integrate with Job Registry**: Ensure the worker interacts with the job registry to fetch job types and execute corresponding tasks. This is done using `job_registry.get(&job.job_type)`.

5. **Logging and Monitoring**: Add logging and monitoring to track the worker's activity and health. This includes logging job execution, errors, and worker status.

Below is a refined example based on the provided `worker.rs` file:

```rust
use crate::job_registry::JobRegistry;
use crate::storage;
use crate::util::{try_to_extract_panic_info, with_sentry_transaction};
use anyhow::anyhow;
use diesel::prelude::*;
use diesel_async::pooled_connection::deadpool::Pool;
use diesel_async::scoped_futures::ScopedFutureExt;
use diesel_async::{AsyncConnection, AsyncPgConnection};
use futures_util::FutureExt;
use sentry_core::{Hub, SentryFutureExt};
use std::panic::AssertUnwindSafe;
use std::sync::Arc;
use std::time::Duration;
use tokio::time::sleep;
use tracing::{debug, error, info_span, warn};

pub struct Worker<Context> {
    pub(crate) connection_pool: Pool<AsyncPgConnection>,
    pub(crate) context: Context,
    pub(crate) job_registry: Arc<JobRegistry<Context>>,
    pub(crate) shutdown_when_queue_empty: bool,
    pub(crate) poll_interval: Duration,
}

impl<Context: Clone + Send + Sync + 'static> Worker<Context> {
    /// Run background jobs forever, or until the queue is empty if `shutdown_when_queue_empty` is set.
    pub async fn run(&self) {
        loop {
            match self.run_next_job().await {
                Ok(Some(_)) => {}
                Ok(None) if self.shutdown_when_queue_empty => {
                    debug!("No pending background worker jobs found. Shutting down the worker…");
                    break;
                }
                Ok(None) => {
                    debug!(
                        "No pending background worker jobs found. Polling again in {:?}…",
                        self.poll_interval
                    );
                    sleep(self.poll_interval).await;
                }
                Err(error) => {
                    error!("Failed to run job: {error}");
                    sleep(self.poll_interval).await;
                }
            }
        }
    }

    /// Run the next job in the queue, if there is one.
    ///
    /// Returns:
    /// - `Ok(Some(job_id))` if a job was run
    /// - `Ok(None)` if no jobs were waiting
    /// - `Err(...)` if there was an error retrieving the job
    async fn run_next_job(&self) -> anyhow::Result<Option<i64>> {
        let context = self.context.clone();
        let job_registry = self.job_registry.clone();
        let mut conn = self.connection_pool.get().await?;

        let job_types = job_registry.job_types();
        conn.transaction(|conn| {
            async move {
                debug!("Looking for next background worker job…");
                let Some(job) = storage::find_next_unlocked_job(conn, &job_types)
                    .await
                    .optional()?
                else {
                    return Ok(None);
                };

                let span = info_span!("job", job.id = %job.id, job.typ = %job.job_type);
                let _enter = span.enter();

                let job_id = job.id;
                debug!("Running job…");

                let future = with_sentry_transaction(&job.job_type, || async {
                    let run_task_fn = job_registry
                        .get(&job.job_type)
                        .ok_or_else(|| anyhow!("Unknown job type {}", job.job_type))?;

                    AssertUnwindSafe(run_task_fn(context, job.data))
                        .catch_unwind()
                        .await
                        .map_err(|e| try_to_extract_panic_info(&e))
                        // TODO: Replace with flatten() once that stabilizes
                        .and_then(std::convert::identity)
                });

                match future.bind_hub(Hub::current()).await {
                    Ok(_) => {
                        debug!("Deleting successful job…");
                        storage::delete_successful_job(conn, job_id).await?
                    }
                    Err(error) => {
                        warn!("Failed to run job: {error}");
                        storage::update_failed_job(conn, job_id).await;
                    }
                }

                Ok(Some(job_id))
            }
            .scope_boxed()
        })
        .await
    }
}
```

This example demonstrates how to create an autonomous worker that continuously processes jobs from a queue, handles errors, retries jobs, and logs its activities.


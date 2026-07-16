# Jobs

- [Introduction](#introduction)
- [Creating a new job](#creating-a-new-job)
- [Register a job](#register-a-job)
- [Overlap prevention](#overlap-prevention)
- [WebSocket integration](#websocket-integration)


## Introduction

Jobs define a way to run periodic background tasks when the server starts. The scheduler uses Go goroutines and tickers, with built-in overlap prevention (a job will not start a new run if the previous one is still executing).


## Creating a new job

Jobs can be placed anywhere, but are recommended to be placed in `app/jobs/`.

Run the CLI command:
```
art make:job CleanupJob
```

This creates a new background job:

```go
package jobs

import (
    "log"
    "github.com/nicklasjeppesen/going_internal/super/channels"
    "github.com/nicklasjeppesen/going_internal/super/jobs"
    "time"
)

func CleanupJob() jobs.Job {
    return jobs.Job{
        Interval:       5 * time.Second,    // required: how often the job runs
        Runner:         Runner,             // required: the function to execute
        Title:          "CleanupJob",       // optional: descriptive name
        TerminateAfter: 10 * time.Second,   // optional: max execution time per run
    }
}

func Runner(job jobs.Job) {
    log.Println("cleanup: start")
    for i := 0; i < 20; i++ {
        if interrupted, err := job.IsInterrupted(); interrupted {
            handleIfJobInterrupted(job)
            return
        }
        time.Sleep(1 * time.Second)
        log.Printf("cleanup: chunk %d done", i+1)
    }
    log.Println("cleanup: job finished")
}

func handleIfJobInterrupted(job jobs.Job) {
    if job.InterruptedByDeadlineExceeded() {
        log.Println("Deadline exceeded")
    } else if job.InterruptedByCanceled() {
        log.Println("Stopped by Ctrl+C")
    }
}
```

### Job context

Each job has its own context that is cancelled when the scheduler shuts down or the deadline is exceeded:

```go
func Runner(job jobs.Job) {
    ctx := job.Ctx()
    select {
    case <-ctx.Done():
        log.Println("Job context cancelled")
        return
    case <-time.After(5 * time.Second):
        log.Println("Work completed")
    }
}
```

### Checking interruption

Use these methods to check why a job was interrupted:

- `job.IsInterrupted()` - Returns true if the job was interrupted for any reason
- `job.InterruptedByDeadlineExceeded()` - Returns true if the `TerminateAfter` deadline was exceeded
- `job.InterruptedByCanceled()` - Returns true if the scheduler was stopped (e.g., via Ctrl+C)


## Register a job

Jobs are registered in `app/jobs/base.go`:

```go
func RegisterJobs(schedule *jobs.Scheduler) {
    schedule.CreateJob(CleanupJob())
}
```


## Overlap prevention

The scheduler prevents overlapping job executions. If a job's `Runner` is still running when the next tick occurs, that tick is skipped. This ensures that long-running jobs don't pile up.


## WebSocket integration

Jobs can send messages to WebSocket clients using the `channels` package:

```go
func Runner(job jobs.Job) {
    for i := 0; i < 10; i++ {
        time.Sleep(1 * time.Second)

        // Send a message to a WebSocket hub
        job.SendMessageToSocket(channels.Socket{
            URL:     "/ws/notifications",
            Message: "progress",
            Data: map[string]any{
                "progress": (i + 1) * 10,
            },
        })
    }
}
```

This triggers the `AppReceiver` method in the hub registered for `/ws/notifications`.

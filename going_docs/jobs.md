
# Jobs

- [Introduction](#introduction)
- [Creating a new job](#creating-a-new-job)
- [Register a job](#register-a-job)



## Introduction
Jobs define a way to run a period jobs in the background when the server starts


## Creating a new job
Jobs can be placed anywhere, but is recomended to be place in `app/jobs/folder`. 

run the cli command: ``` art make:job CleanupJob ```
This will createa a new background job
The jobs print integer from 1-20.

```go
package jobs

import (
	"log"
	"myapp/internal/super/channels"
	"myapp/internal/super/jobs"
	"time"
)

func CleanupJob() Job {
    return jobs.Job{
        Interval:       5 * time.Second,    // required
        Runner:         Runner,             // required
        Title:          "CleanJob",         // Optional
        // TerminateAfter: 10 * time.Second, // Optional
	}
}

func Runner(job Job) {
	log.Println("cleanup: start")
	for i := 0; i < 20; i++ {
		if job.IsInterrupted() {
			HandleIfJobInterrupted(job)
			return
		}
		time.Sleep(1 * time.Second)
		log.Printf("cleanup: chunk %d done", i+1)
	}
	log.Println("CLEANUP: Job finised")
}

func HandleIfJobInterrupted(job Job) {
	if job.InterruptedByDeadlineExceeded() {
		log.Println("OH no, exceed Deadline limit")
	} else if job.InterruptedByCanceled() {
		log.Println("cleanup: Stop by ctr c command")
	}
}
```

<a name="Register-job"></a>
## Register a job

Jobs registered here: `app/jobs/base.go`  like this: 


```go
func RegisterJobs(schedule *jobs.Scheduler) {
	schedule.CreateJob(CleanupJob()) // new job
}
```



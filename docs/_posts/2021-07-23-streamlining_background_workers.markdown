---
layout: post
title:  "Streamlining Background Workers"
date:   2021-07-23 12:38:13 -0400
---

As with every rapidly-scaling startup, technical debt from hasty deploys and legacy practices piles up quickly. We soon found our suite of background workers to be in a disarray.

We were supporting two background job frameworks - legacy workers tended to be on Resque, whereas newer workers tended to be on Sidekiq. Resque jobs tended to be slower, inefficient, and backed up the queue. Most disturbingly, we found that we had no visibility over the jobs running on Resque. Jobs would routinely fail and disappear altogether without proper visibility or warning - **we were often in the dark as to what and why things went wrong**.

The goal was to streamline our background job workers and set a golden standard moving forward. This article will showcase our learnings and illustrate the process and guidelines we found to be optimal.

**Deciding on a framework**

First off, let's differentiate between Sidekiq and Resque.

Sidekiq is thread based - meaning Sidekiq will compile your application code once and use multiple threads to process multiple jobs.

Resque is process based - meaning Resque will compile a copy of your application code for each and every one of its worker processes. This tends to take up more memory and more compile time with each process. More info [here][source-1] and [here][source-2].

Due to its better support, features, and efficiency, we quickly decided to adopt Sidekiq Enterprise as our primary and sole background framework.

We chose not to utilize Rails6 ActiveJob framework, as pure Sidekiq is more performant in processing and may have compatibility issues. More info [here][source-3] and [here][source-4].

**Queues: Less = More Performance**

Previously, we were creating new queues based on job types and domains. On Resque, our application was running over 50 unique queues.

We found our Sidekiq dynos to be more performant with less queues. So instead of managing over 50 queues, which in and of itself is troublesome and problematic, we streamlined our workers into 4 queues based on SLA times.

**Critical** - These jobs should run ASAP. The essentials - think of mailers, confirmations, payment processing. We set the latency SLA to be under one minute.

**Default -** These jobs should run soonish. We set the latency here to be 60 minutes. The bulk of your jobs ~80% should be in this queue.

**Low -** These jobs should run eventually. We set the latency here to be under a day.

**Backfill -** Sometimes we have huge backfill jobs that queue up hundreds and thousands of workers immediately - we made a separate queue to process those jobs as to not backup the previous queues.

So instead of staring at a huge nonsensical Procfile, we now have a really clean and intuitive queue strategy for new and existing jobs.

**Dynos: Perf-L dynos > Perf-M dynos**

Performance L dynos ($500) cost twice as much as Performance M dynos ($250), but offer ~4 times the resources (12x vs 50x) - meaning they're twice the bang per buck.

This means we should try to run multiple Sidekiq processes on a performance L dyno first, rather than trying to scale and add on multiple performance M dynos.

**Workers: Idempotent and Small**

**Idempotent** - meaning we can run the workers repeatedly w/o bad results. This'll cut down on headaches down the road should unintended results arise from erroneous retries.

```jsx
# BAD
class SendDMCAEmailWorker
  def perform(user_id)
    user = User.find(user_id)
    user.notify
  end
end

# GOOD
class SendDMCAEmailWorker
  def perform(user_id)
    user = User.find(user_id)
    user.notify unless user.notified?
  end
end
```

**Small** -  Workers that take a long to time to execute clog up the queue and cause other jobs to take too long to execute. A job should be responsible for one of two things: the orchestration of a batch of jobs, or the execution of a single job.

In the case of a worker that takes a long time to execute, our goal is to split up the main job into multiple smaller jobs and utilize parallelization if possible. Rather than have one worker process (lets say 10 seconds) one thing for ten users (100 seconds), we want to spawn ten jobs to process one thing for one user (10 seconds!).

```jsx
# BAD

class XYZWorker
  def perform(user_ids)
    user_ids.each do |user_id|
      user = User.find(user_id)
      user.process # work work work
    end
  end
end

# GOOD

class XYZQueuerWorker
  def perform(user_ids)
    user_ids.each do |user_id|
      XYZWorker.perform_async(user_id: user_id)
    end
  end
end

class XYZWorker
  def perform(user_id)
    user = User.find(user_id)
    summary = user.process
  end
end
```

**Bringing it all together - Visibility, Reliability, and Scalability**

The final piece of our background jobs overhaul - we cared specifically about seeing metrics on job performance, success rates, and failure rates at a moment's glance.

There are a myriad of reporting tools out there - we chose a combination of in-house tools, NewRelic, and Bugsnag. Once we were able to get metrics in, we hooked in pagerduty alerts on critical statuses:  queue latency, queue backfill size, job failure rate - ex: if the critical queue gets backed up above a certain threshold, we want to be alerted immediately to remedy that.

**Conclusion**

It's hard to quantify numerically the effectiveness of all these improvements, but we were able to audit and migrate all our existing jobs from Resque onto Sidekiq and *see* performance increases on Sidekiq despite doubling the workload. Perhaps more importantly, we realized decoding and decreasing the complexity of our system of itself has countless implicit benefits for engineering happiness.

This was a huge effort paved with weeks of research, investment, and pull requests - but the results were undeniable. **We now had an intuitive, efficient, and scalable standard for background workers and jump-started a culture of platform visibility!**

Let me know if this framework helps supplement your engineering processes!

[source-1]: https://dev.to/molly/switching-from-resque-to-sidekiq-3b04
[source-2]: https://joshrendek.com/2012/11/sidekiq-vs-resque/
[source-3]: https://github.com/mperham/sidekiq/wiki/Active-Job#performance
[source-4]: https://github.com/mperham/sidekiq/wiki/Active-Job#commercial-features

---
layout: post
title:  "Testing Architecture: Heroku Pipelines and Data"
date:   2021-08-29 06:04:13 -0400
---

### Staging Environments

For a long time, a single shared staging environment was sufficient enough for the handful of devs on our engineering team. However, as the startup matured and more and more developers joined the team, we quickly encountered two problems.

1. The single staging environment quickly became a bottleneck.

   Developers would sit idly by, waiting for their turn to deploy and test their feature.

    Our #dev-staging often looked like this -

    > using staging for an hour

    > ill jump on after for 20, thanks

    > hey, you still on staging?

    > hey sorry go for it

    > no worries

    > hey lmk when you’re done. no rush

2. The staging environment itself was unkempt and unreliable, for a myriad of reasons:
    - the data was old - staging's database used to be a 1:1 copy of prod's, but the production database quickly became too large for staging to keep up. Eventually our jobs to sync our staging database with production's simply timed out, relegating our staging database with old data.
    - the data was constantly changing - Any changes one developer made onto the environment often persisted for everyone else. Any testing profiles could potentially be overridden and ruined by another developer.
    - faulty migrations -  if one were to run a new migration and did not revert it (as it happens to best of us), the faulty migration would persist and would most likely setup an unpleasant experience for the next developer.

So the question we found ourselves asking was: **How do we create a more reliable testing environment for our developers?**

---

### Shared vs Separate

The straightforward, easy solution with the least investment was to simply create another shared staging environment.

This solution works in a pinch, and I know of some very established startups and companies that are still on this format. The drawback however, is that since the approach scales horizontally, problems one and two actually get worse as the organization gets bigger and bigger. Not only would you have to keep track of who's using which environment, but the environments themselves become difficult to keep track.

We wanted to take this opportunity to invest and build a solution that was maintainable and scalable. We also wanted to invest in a solution that **let each developer have their own containerized testing environment**, to avoid the problems listed above.

---

### Review Apps

We quickly narrowed our answer to review apps. Review Apps are disposable, encapsulated, and configurable Heroku apps. They can be set to spawn and sync with a pull request allowing developers to continuously test and share their code.

As most of our infrastructure was hosted on Heroku, it made logical sense to utilize Heroku Review Apps, then go custom or third party, ie: [https://releasehub.com/](https://releasehub.com/).

Heroku has extensive documentation on setting up review apps [here](source-3), and the gist of it boils down to a file called app.json. App.json tells Heroku how to create a review-app: we can specify what ENVs, add-ons, dyno sizes, and even postdeploy scripts to add on a review app.

Here's an example of a app.json

```ruby
{
  "name": "review",
  "repository": "https://github.com/jane-doe/small-sharp-tool",
  "scripts": {
    "postdeploy": "bundle exec rake db:migrate"
  },
  "env": {
    "WEB_CONCURRENCY": {
      "description": "The number of processes to run.",
      "value": "5"
    },
    "MAX_THREADS": {
      "value": "1"
    }
  },
  "formation": {
    "web": {
      "quantity": 1,
      "size": "standard-1x"
    }
  },
  "image": "heroku/ruby",
  "addons": [
    {
      "plan": "heroku-postgresql",
      "options": {
        "version": "9.5"
      }
    },
    "redistogo:nano",
    "sendgrid:starter"
  ],
  "buildpacks": [
    { "url": "heroku/ruby" },
  ],
  "environments": {
    "test": {
      "scripts": {
        "test": "bundle exec rake test"
      }
    }
  }
}
```
Setting up the app.json and consequently the review app was pretty straightforward. The problem we soon ran into was how to properly fill our review app with data.

---

### Data

Heroku recommends to seed review apps with seed scripts [like so.][source-1]

However, given the nature of our data, testers needed data as close to prod's for practicality. We opted to build our review apps from prod.

Our production database has terabytes on terabytes of data. Replicating that data for a testing environment is not feasible. **So we had to figure out how to take a snapshot of production, have it be as small as possible, all the while having sufficient data for the application to compile and the snapshot to be useful for developer testing.**

The answer came from slowly built up a domain mapping of important tables and their dependencies by talking to various domain experts across search, data, and engineering.

Once we had that domain map, we utilized a modified version of a gem [Brillo][source-2] that given a YAML file, allows one to specify how and which tables to pull in, as well as which data to obfuscate.

Here's an example of a brillo.yml

```ruby
name: review_database
compress: false
transfer:
  enabled: true
  bucket: review_uploads
  region: us-east-1
explore:
  countries:
    tactic: all
    associations:
      shipping_codes:
  shipping/method:
    tactic: delegate_to_model
    associations:
      shipping_price_ranges:
  users:
      tactic: all
obfuscations:
  users.email: email
  users.phone: email

```

Through many iterations and trials, we were able to meticulously slice and dice down the snapshot size from TBs to under 50mb.

We loaded that snapshot into S3, and set up a job to routinely update it from prod.

Because of how easy it is to tell Brillo to grab new tables, keeping relevant data in staging becomes much less of a chore.

Once we built a functional snapshot for review apps, we carefully carefully configured all the necessary add-on services: redis, elasticsearch, mailers, external services and endpoints, to  replicate a feature-rich testing environment.

---

### Conclusion

Developers can now spawn a configurable testing sandbox with a single click, and continuously iterate, test and deploy with every commit.

Review apps with a minimum viable data set was our approach to scaling staging environments, and it worked out splendidly.

As always, let me know your thoughts and if this article helped in any way!

[source-1]: https://help.heroku.com/LBQQH5UZ/copy-database-to-review-app
[source-2]: https://github.com/bessey/brillo
[source-3]: https://devcenter.heroku.com/articles/github-integration-review-apps



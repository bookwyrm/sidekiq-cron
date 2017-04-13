Sidekiq-Cron [![Gem Version](https://badge.fury.io/rb/sidekiq-cron.svg)](http://badge.fury.io/rb/sidekiq-cron) [![Build Status](https://travis-ci.org/ondrejbartas/sidekiq-cron.svg?branch=master)](https://travis-ci.org/ondrejbartas/sidekiq-cron) [![Coverage Status](https://coveralls.io/repos/ondrejbartas/sidekiq-cron/badge.svg?branch=master)](https://coveralls.io/r/ondrejbartas/sidekiq-cron?branch=master)
================================================================================================================================================================================================================================================================================================================================================================================================================================================

[![Join the chat at https://gitter.im/ondrejbartas/sidekiq-cron](https://badges.gitter.im/ondrejbartas/sidekiq-cron.svg)](https://gitter.im/ondrejbartas/sidekiq-cron?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)

[Introduction video about Sidekiq-Cron by Drifting Ruby](https://www.driftingruby.com/episodes/periodic-tasks-with-sidekiq-cron)

A scheduling add-on for [Sidekiq](http://sidekiq.org).

Runs a thread alongside Sidekiq workers to schedule jobs at specified times (using cron notation `* * * * *` parsed by [Rufus-Scheduler](https://github.com/jmettraux/rufus-scheduler), more about [cron notation](http://www.nncron.ru/help/EN/working/cron-format.htm).

Checks for new jobs to schedule every 10 seconds and doesn't schedule the same job multiple times when more than one Sidekiq worker is running.

Scheduling jobs are added only when at least one Sidekiq process is running.

If you want to know how scheduling work, check out [under the hood](#under-the-hood)

Works with ActiveJob (Rails 4.2+)

You don't need Sidekiq PRO, you can use this gem with plain __Sidekiq__.

Requirements
-----------------

- Redis 2.8 or greater is required. (Redis 3.0.3 or greater is recommended for large scale use)
- Sidekiq 4 or greater is required (for Sidekiq < 4 use version sidekiq-cron 0.3.1)

Change Log
----------
before upgrading to new version, please read:
[Change Log](https://github.com/ondrejbartas/sidekiq-cron/blob/master/Changes.md)

Installation
------------

    $ gem install sidekiq-cron

or add to your `Gemfile`

    gem "sidekiq-cron", "~> 0.4.0"


Getting Started
-----------------


If you are not using Rails, you need to add `require 'sidekiq-cron'` somewhere after `require 'sidekiq'`.

_Job properties_:

```ruby
{
 'name'  => 'name_of_job', #must be uniq!
 'cron'  => '1 * * * *',
 'class' => 'MyClass',
 #OPTIONAL
 'queue' => 'name of queue',
 'args'  => '[Array or Hash] of arguments which will be passed to perform method',
 'active_job' => true,  # enqueue job through rails 4.2+ active job interface
 'queue_name_prefix' => 'prefix', # rails 4.2+ active job queue with prefix
 'queue_name_delimiter' => '.'  # rails 4.2+ active job queue with custom delimiter
}
```

### Time, cron and sidekiq-cron

sidekiq-cron uses [rufus-scheduler](https://github.com/jmettraux/rufus-scheduler) to parse the cronline.
By default, the timezone this is evaluated against UTC.
If you want to have your jobs enqueued based on a different time zone you can specify a timezone in the cronline,
like this `'0 22 * * 1-5 America/Chicago'`.
See [rufus-scheduler documentation](https://github.com/jmettraux/rufus-scheduler#a-note-about-timezones) for more information.

### What objects/classes can be scheduled
#### Sidekiq Worker
In this example, we are using `HardWorker` which looks like:
```ruby
class HardWorker
  include Sidekiq::Worker
  def perform(*args)
    # do something
  end
end
```

#### Active Job Worker
You can schedule: `ExampleJob` which looks like:
```ruby
class ExampleJob < ActiveJob::Base
  queue_as :default

  def perform(*args)
    # Do something
  end
end
```

#### Adding Cron job:
```ruby

class HardWorker
  include Sidekiq::Worker
  def perform(name, count)
    # do something
  end
end

Sidekiq::Cron::Job.create(name: 'Hard worker - every 5min', cron: '*/5 * * * *', class: 'HardWorker')
# => true
```

`create` method will return only true/false if job was saved or not.

```ruby
job = Sidekiq::Cron::Job.new(name: 'Hard worker - every 5min', cron: '*/5 * * * *', class: 'HardWorker')

if job.valid?
  job.save
else
  puts job.errors
end

#or simple

unless job.save
  puts job.errors #will return array of errors
end
```

Load more jobs from hash:
```ruby

hash = {
  'name_of_job' => {
    'class' => 'MyClass',
    'cron'  => '1 * * * *',
    'args'  => '(OPTIONAL) [Array or Hash]'
  },
  'My super iber cool job' => {
    'class' => 'SecondClass',
    'cron'  => '*/5 * * * *'
  }
}

Sidekiq::Cron::Job.load_from_hash hash
```

Load more jobs from array:
```ruby
array = [
  {
    'name'  => 'name_of_job',
    'class' => 'MyClass',
    'cron'  => '1 * * * *',
    'args'  => '(OPTIONAL) [Array or Hash]'
  },
  {
    'name'  => 'Cool Job for Second Class',
    'class' => 'SecondClass',
    'cron'  => '*/5 * * * *'
  }
]

Sidekiq::Cron::Job.load_from_array array
```

Bang-suffixed methods will remove jobs that are not present in the given hash/array,
update jobs that have the same names, and create new ones when the names are previously unknown.

```ruby
Sidekiq::Cron::Job#load_from_hash! hash
Sidekiq::Cron::Job#load_from_array! array
```

or from YML (same notation as Resque-scheduler)
```yaml
#config/schedule.yml

my_first_job:
  cron: "*/5 * * * *"
  class: "HardWorker"
  queue: hard_worker

second_job:
  cron: "*/30 * * * *"
  class: "HardWorker"
  queue: hard_worker_long
  args:
    hard: "stuff"
```

```ruby
#initializers/sidekiq.rb
schedule_file = "config/schedule.yml"

if File.exists?(schedule_file) && Sidekiq.server?
  Sidekiq::Cron::Job.load_from_hash YAML.load_file(schedule_file)
end
```

#### Finding jobs
```ruby
#return array of all jobs
Sidekiq::Cron::Job.all

#return one job by its unique name - case sensitive
Sidekiq::Cron::Job.find "Job Name"

#return one job by its unique name - you can use hash with 'name' key
Sidekiq::Cron::Job.find name: "Job Name"

#if job can't be found nil is returned
```

#### Destroy jobs:
```ruby
#destroys all jobs
Sidekiq::Cron::Job.destroy_all!

#destroy job by its name
Sidekiq::Cron::Job.destroy "Job Name"

#destroy found job
Sidekiq::Cron::Job.find('Job name').destroy
```

#### Work with job:
```ruby
job = Sidekiq::Cron::Job.find('Job name')

#disable cron scheduling
job.disable!

#enable cron scheduling
job.enable!

#get status of job:
job.status
# => enabled/disabled

#enqueue job right now!
job.enque!
```

How to start scheduling?
Just start Sidekiq workers by running:

    sidekiq

### Web UI for Cron Jobs

If you are using Sidekiq's web UI and you would like to add cron jobs too to this web UI,
add `require 'sidekiq/cron/web'` after `require 'sidekiq/web'`.

With this, you will get:
![Web UI](https://github.com/ondrejbartas/sidekiq-cron/raw/master/examples/web-cron-ui.png)

### Forking Processes

If you're using a forking web server like Unicorn you may run into an issue where the Redis connection is used
before the process forks, causing the following exception

    Redis::InheritedError: Tried to use a connection from a child process without reconnecting. You need to reconnect to Redis after forking.

to occcur. To avoid this, wrap your job creation in the call to `Sidekiq.configure_server`:

```ruby
Sidekiq.configure_server do |config|
  schedule_file = "config/schedule.yml"

  if File.exists?(schedule_file)
    Sidekiq::Cron::Job.load_from_hash YAML.load_file(schedule_file)
  end
end
```

Note that this API is only available in Sidekiq 3.x.x.

## Under the hood

When you start the Sidekiq process, it starts one thread with `Sidekiq::Poller` instance, which perform the adding of scheduled jobs to queues, retries etc.

Sidekiq-Cron adds itself into this start procedure and starts another thread with `Sidekiq::Cron::Poller` which checks all enabled Sidekiq cron jobs every 10 seconds, if they should be added to queue (their cronline matches time of check).


## Thanks to
* [@7korobi](https://github.com/7korobi)
* [@antulik](https://github.com/antulik)
* [@felixbuenemann](https://github.com/felixbuenemann)
* [@gstark](https://github.com/gstark)
* [@RajRoR](https://github.com/RajRoR)
* [@romeuhcf](https://github.com/romeuhcf)
* [@siruguri](https://github.com/siruguri)
* [@Soliah](https://github.com/Soliah)
* [@stephankaag](https://github.com/stephankaag)
* [@sue445](https://github.com/sue445)
* [@sylg](https://github.com/sylg)
* [@tmeinlschmidt](https://github.com/tmeinlschmidt)
* [@zerobearing2](https://github.com/zerobearing2)


## Contributing to sidekiq-cron

* Check out the latest master to make sure the feature hasn't been implemented or the bug hasn't been fixed yet.
* Check out the issue tracker to make sure someone already hasn't requested it and/or contributed it.
* Fork the project.
* Start a feature/bugfix branch.
* Commit and push until you are happy with your contribution.
* Make sure to add tests for it. This is important so I don't break it in a future version unintentionally.
* Please try not to mess with the Rakefile, version, or history. If you want to have your own version, or is otherwise necessary, that is fine, but please isolate to its own commit so I can cherry-pick around it.

## Copyright

Copyright (c) 2013 Ondrej Bartas. See LICENSE.txt for further details.

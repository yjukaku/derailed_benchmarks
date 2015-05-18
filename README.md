## Derailed Benchmarks

A series of things you can use to benchmark a Rails or Ruby app.

![](http://media.giphy.com/media/lfbxexWy71b6U/giphy.gif)

## Compatibility/Requirements

This gem has been tested and is known to work with Rails 3.2+ using Ruby
2.1+. Some commands __may__ work on older versions of Ruby, but not all commands are supported.

For some benchmarks, not all, you'll need to verify you have a working version of curl on your OS:

```
$ which curl
/usr/bin/curl
$ curl -V
curl 7.37.1 #...
```

## Install

Put this in your gemfile:

```
gem 'derailed', group: :development
```

Then run `$ bundle install`.

While executing your commands you may need to use `bundle exec` before typing the command.

To use all profiling methods available also add:

```
gem "stackprof", group: :development
```

You must be using Ruby 2.1+ to install these libraries. If you're on an older version of Ruby, what are you waiting for?

## Use

There are two ways to use benchmark an app. Derailed can either try to boot your web app and run requests against it while benchmarking, or it can also staically give you more information about your dependencies that are in your your Gemfile. Booting your app will always be more accurate, but if you cannot get your app to run in production locally, you'll still find the static information useful.

## Static Benchmarking

This section covers how to get memory information from your Gemfile without having to boot your app.

All commands in this section will begin with `$ derailed bundle:`

For more information on the relationship between memory and performance please read/watch [How Ruby Uses Memory](http://www.schneems.com/2015/05/11/how-ruby-uses-memory.html).

### Memory used at Require time

Each gem you add to your project can increase your memory at boot. You can get visibility into the total memory used by each gem in your Gemfile by running:

```
$ derailed bundle:mem
```

This will load each of your gems in your Gemfile and see how much memory they consume when they are required. For example if you're using the `mail` gem. The output might look like this

```
$ derailed bundle:mem
TOP: 54.1836 MiB
  mail: 18.9688 MiB
    mime/types: 17.4453 MiB
    mail/field: 0.4023 MiB
    mail/message: 0.3906 MiB
  action_view/view_paths: 0.4453 MiB
    action_view/base: 0.4336 MiB
```

_Aside: A "MiB", which is the [IEEE] and [IEC] symbol for Mebibyte, is 2<sup>20</sup> bytes / 1024 Kibibytes (which are in turn 1024 bytes)._

[IEEE]: http://en.wikipedia.org/wiki/IEEE_1541-2002
[IEC]: http://en.wikipedia.org/wiki/IEC_80000-13

Here we can see that `mail` uses 18MiB, with the majority coming from `mime/types`. You can use this information to prune out large dependencies you don't need. Also if you see a large memory use by a gem that you do need, please open up an issue with that library to let them know (be sure to include reproduction instructions). Hopefully as a community we can identify memory hotspots and reduce their impact. Before we can fix performance problems, we need to know where those problems exist.

By default this task will only return results from the `:default` and `"production"` groups. If you want a different group you can run with.

```
$ derailed bundle:mem development
```

You can use `CUT_OFF=0.3` to only show files that have above a certain memory useage, this can be used to help eliminate noise.

Note: This method won't include files in your own app, only items in your Gemfile. For that you'll need to use `derailed exec mem`. See below for more info.

### Objects created at Require time

To get more info about the objects, using [memory_profiler](https://github.com/SamSaffron/memory_profiler), created when your dependencies are required you can run:

```
$ derailed bundle:objects
```

This will output detailed information about objects created while your dependencies are loaded

```
Measuring objects created by gems in groups [:default, "production"]
Total allocated 433895
Total retained 100556

allocated memory by gem
-----------------------------------
  24369241  activesupport-4.2.1
  15560550  mime-types-2.4.3
   8103432  json-1.8.2
```

Once you identify a gem that creates a large amout of memory using `$ derailed bundle:mem` you can pull that gem into it's own Gemfile and run `$ derailed bundle:objects` to get detailed information about it. This information can be used by contributors and library authors to identify and eliminate object creation hotspots.


By default this task will only return results from the `:default` and `"production"` groups. If you want a different group you can run with.

```
$ derailed bundle:objects development
```

Note: This method won't include files in your own app, only items in your Gemfile. For that you'll need to use `derailed exec objects`. See below for more info.


## Dynamic app Benchmarking

This benchmarking will attempt to boot your Rails app and run benchmarks against it. Unlike the static benchmarking with `$ derailed bundle:*` these will include information about your specific app. The pro is you'll get more information and potentially identify problems in your app code, the con is that it requires you to be able to boot and run your application in a `production` environment locally, which for some apps is non-trivial.

You may want to check out [mini-profiler](https://github.com/MiniProfiler/rack-mini-profiler), here's a [mini-profiler walkthrough](http://www.justinweiss.com/blog/2015/05/11/a-new-way-to-understand-your-rails-apps-performance/?utm_source=rubyweekly&utm_medium=email). It's great and does slightly different benchmarking than what you'll find here.

### Running in Production Locally.

Before you want to attempt any dynamic benchmarks, you'll need to boot your app in `production` mode. We benchmark using `production` because it is close to your deployed performance. This section is more a collection of tips rather than a de-facto tutorial.

For starters try booting into a console:

```
$ RAILS_ENV=production rails console
```

You may get errors, complaining about not being able to connect to the `production` database. For this, you can either create a local database with the name of your production database, or you can copy the info from your `development` group to your `production` groupu in your `database.yml`.

You may be missing environment variables expected in `production` such as `SECRET_KEY_BASE`. For those you can either commit them to your `.env` file (if you're using one). Or add them directly to the command:

```
$ SECRET_KEY_BASE=foo RAILS_ENV=production rails console
```

Once you can boot a console in production, you'll need to be able to boot a server in production

```
$ RAILS_ENV=production rails server
```

You may need to disable enforcing SSL or other domain restrictions temporarially. If you do these, don't forget to add them back in before deploying any code (eek!).

You can get information from STDOUT if you're using `rails_12factor` gem, or from `logs/production.log` by running

```
$ tail -f logs/production.log
```

Once you've fixed all errors and you can run a server in production, you're almost there.

### Running Derailed Exec

You can run commands against your app by running `$ derailed exec`. There are sections on setting up Rack and using authenticated requests below. You can see what commands are available by running:

```
$ derailed exec --help
  $ derailed exec perf:allocated_objects  # outputs allocated object diff after app is called TEST_COUNT times
  $ derailed exec perf:gc  # outputs GC::Profiler.report data while app is called TEST_COUNT times
  $ derailed exec perf:ips  # iterations per second
  $ derailed exec perf:mem  # show memory usage caused by invoking require per gem
  $ derailed exec perf:objects  # profiles ruby allocation
  $ derailed exec perf:ram_over_time  # outputs ram usage over time
  $ derailed exec perf:test  # hits the url TEST_COUNT times
```

Instead of going over each command we'll look at common problems and which commands are best used to diagnose them. Later on we'll cover all of the environment variables you can use to configure derailed benchmarks in it's own section.


### Is my app leaking memory?

If your app appears to be leaking ever increasing amounts of memory, you'll want to first verify if it's an actual unbound "leak" or if it's just using more memory than you want. A true memory leak will increase memory use forever, most apps will increase memory use until they hit a "plateau". To diagnose this you can run:

```
$ derailed exec perf:ram_over_time
```

This will boot your app and hit it with requests and output the memory to stdout (and a file under ./tmp). It may look like this:

```
$ derailed exec perf:ram_over_time
Booting: production
Endpoint: "/"
PID: 78675
103.55078125
178.45703125
179.140625
180.3671875
182.1875
182.55859375
# ...
183.65234375
183.26171875
183.62109375
```

Here we can see that while the memory use is increasing, it levels off around 183 MiB. You'll want to run this task using ever increasing values of `TEST_COUNT=` for example

```
$ TEST_COUNT=5000 derailed exec perf:ram_over_time
$ TEST_COUNT=10_000 derailed exec perf:ram_over_time
$ TEST_COUNT=20_000 derailed exec perf:ram_over_time
```

Adjust your counts appropriately so you can get results in a reasonable amount of time. If your memory never levels off, congrats! You've got a memory leak! I recommend copying and pasting values from the file generated into google docs and graphing it so you can get a better sense of the slope of your line.

If you don't want it to generate a tmp file with results run with `SKIP_FILE_WRITE=1`.

If you're pretty sure that there's a memory leak, but you can't confirm it using this method. Look at the environment variable options below, you can try hitting a different endpoint etc.

## Dissecting a Memory Leak

If you've identified a memory leak, or you simply want to see where your memory use is coming from you'll want to use

```
$ derailed exec perf:objects
```

This task hits your app and uses memory_profiler to see where objects are created. You'll likely want to run once, then run it with a higher `TEST_COUNT` so that you can see hotspots where objects are created on __EVERY__ request versus just maybe on the first.


```
$ TEST_COUNT=10 derailed exec perf:objects
```

This is an expensive operation, so you likely want to keep the count lowish. Once you've identified a hotspot read [how ruby uses memory](http://www.sitepoint.com/ruby-uses-memory/) for some tips on reducing object allocations.

This is is similar to `$ derailed bundle:objects` however it includes objects created at runtime. It's much more useful for actual production performance debugging, the other is more useful for library authors to debug.


### Memory Is large at boot.

Ruby memory typically goes in one direction, up. If your memory is large when you boot the application it will likely only increase. In addition to debugging memory retained from dependencies obtained while running `$ derailed bundle:mem` you'll likely want to see how your own files contribute to memory use.

This task does essentially the same thing, however it hits your app with one request to ensure that any last minute `require`-s have been called. To execute you can run:


```
$ derailed exec perf:mem

TOP: 54.1836 MiB
  mail: 18.9688 MiB
    mime/types: 17.4453 MiB
    mail/field: 0.4023 MiB
    mail/message: 0.3906 MiB
  action_view/view_paths: 0.4453 MiB
    action_view/base: 0.4336 MiB
```

You can use `CUT_OFF=0.3` to only show files that have above a certain memory useage, this can be used to help eliminate noise.

If your application code is exremely large at boot consider using `$ derailed exec perf:objects` to debug low level object creation.

## My app is Slow

Well...aren't they all. If you've already looked into decreasing object allocations, you'll want to look at where your app is spending the most amount of code. Once you know that, you'll know where to spend your time optimising.

One technique is to use a "sampling" stack profiler. This type of profiling looks at what method is being executed at a given interval and records it. At the end of execution it counts all the times a given method was being called and shows you the percent of time spent in each method. This is a very low overhead method to looking into execution time. Ruby 2.1+ has this available in gem form it's called [stackprof](https://github.com/tmm1/stackprof). As you guessed you can run this with derailed benchmarks, first add it to your gemfile `gem "stackprof", group: :development` then execute:

```
$ derailed exec perf:stackprof
==================================
  Mode: cpu(1000)
  Samples: 16067 (1.07% miss rate)
  GC: 2651 (16.50%)
==================================
     TOTAL    (pct)     SAMPLES    (pct)     FRAME
      1293   (8.0%)        1293   (8.0%)     block in ActionDispatch::Journey::Formatter#missing_keys
       872   (5.4%)         872   (5.4%)     block in ActiveSupport::Inflector#apply_inflections
       935   (5.8%)         802   (5.0%)     ActiveSupport::SafeBuffer#safe_concat
       688   (4.3%)         688   (4.3%)     Temple::Utils#escape_html
       578   (3.6%)         578   (3.6%)     ActiveRecord::Attribute#initialize
      3541  (22.0%)         401   (2.5%)     ActionDispatch::Routing::RouteSet#url_for
       346   (2.2%)         346   (2.2%)     ActiveSupport::SafeBuffer#initialize
       298   (1.9%)         298   (1.9%)     ThreadSafe::NonConcurrentCacheBackend#[]
       227   (1.4%)         227   (1.4%)     block in ActiveRecord::ConnectionAdapters::PostgreSQLAdapter#exec_no_cache
       218   (1.4%)         218   (1.4%)     NewRelic::Agent::Instrumentation::Event#initialize
      1102   (6.9%)         213   (1.3%)     ActiveSupport::Inflector#apply_inflections
       193   (1.2%)         193   (1.2%)     ActionDispatch::Routing::RouteSet::NamedRouteCollection::UrlHelper#deprecate_string_options
       173   (1.1%)         173   (1.1%)     ActiveSupport::SafeBuffer#html_safe?
       308   (1.9%)         171   (1.1%)     NewRelic::Agent::Instrumentation::ActionViewSubscriber::RenderEvent#metric_name
       159   (1.0%)         159   (1.0%)     block in ActiveRecord::Result#hash_rows
       358   (2.2%)         153   (1.0%)     ActionDispatch::Routing::RouteSet::Generator#initialize
       153   (1.0%)         153   (1.0%)     ActiveRecord::Type::String#cast_value
       192   (1.2%)         143   (0.9%)     ActionController::UrlFor#url_options
       808   (5.0%)         127   (0.8%)     ActiveRecord::LazyAttributeHash#[]
       121   (0.8%)         121   (0.8%)     PG::Result#values
       120   (0.7%)         120   (0.7%)     ActionDispatch::Journey::Router::Utils::UriEncoder#escape
      2478  (15.4%)         117   (0.7%)     ActionDispatch::Journey::Formatter#generate
       115   (0.7%)         115   (0.7%)     NewRelic::Agent::Instrumentation::EventedSubscriber#event_stack
       114   (0.7%)         114   (0.7%)     ActiveRecord::Core#init_internals
       263   (1.6%)         110   (0.7%)     ActiveRecord::Type::Value#type_cast
      8520  (53.0%)         102   (0.6%)     ActionView::CompiledTemplates#_app_views_repos__repo_html_slim__2939326833298152184_70365772737940
```

From here you can dig into individual methods.

## Is this perf change faster?

Micro benchmarks might tell you at the code level how much faster something is, but what about the overall application speed. If you're trying to figure out how effective a performance change is to your application, it is useful to compare it to your existing app performance. To help you with that you can use:

```
$ derailed exec perf:ips
Endpoint: "/"
Calculating -------------------------------------
                 ips     1.000  i/100ms
-------------------------------------------------
                 ips      3.306  (± 0.0%) i/s -     17.000
```

This will hit an endpoint in your application using [benchmark-ips](https://github.com/evanphx/benchmark-ips). In "iterations per second" a higher value is always better. You can run your code change several times using this method, and then run your "baseline" codebase (without your changes) to see how it affects your overall performance. You'll want to run and record the results several times (including the std deviation) so you can help eliminate noise. Benchmarking is hard, this technique isn't perfect but it's definetly better than nothing.

If you care you can also run pure benchmark (without ips):

```
$ derailed exec perf:test
```

But I wouldn't, benchmark-ips is a better measure.


## Environment Variables

All the tasks accept configuration in the form of environment variables.


### Increasing or decreasing test count `TEST_COUNT`

For tasks that are run a number of times you can set the number using `TEST_COUNT` for example:

```
$ derailed exec perf:test TEST_COUNT=100_000
```

## Hitting a different endpoint with `PATH_TO_HIT`

By default tasks will hit your homepage `/`. If you want to hit a different url use `PATH_TO_HIT` for example if you wanted to go to `users/new` you can execute:

```
$ PATH_TO_HIT=/users/new derailed exec perf:mem
```

### Using a real web server with USE_SERVER

All tests are run without a webserver (directly using `Rack::Mock` by default), if you want to use a webserver set `USE_SERVER` to a Rack::Server compliant server, such as `webrick`.

```
$ USE_SERVER=webrick derailed exec perf:mem
```

Or

```
$ USE_SERVER=puma derailed exec perf:mem
```

This boots a webserver and hits it using `curl` instead of in memory. This is useful if you think the performance issue is related to your webserver.

Note: this plugs in the given webserver directly into rack, it doesn't use any `puma.config` file etc. that you have set-up. If you want to do this, i'm open to suggestions on how (and pull requests)


### Running in a different environment

Tests run against the production environment by default, but it's easy to
change this if your app doesn't run locally with `RAILS_ENV` set to
`production`. For example:

```
$ derailed exec perf:mem RAILS_ENV=development
```

## perf.rake

If you want to customize derailed, you'll need to create a `perf.rake` file at the root of the directory you're trying to benchmark.

It is possible to run benchmarks directly using rake

```
$ cat <<  EOF > perf.rake
  require 'bundler'
  Bundler.setup

  require 'derailed_benchmarks'
  require 'derailed_benchmarks/tasks'
EOF
```

The file should look like this:

```
$ cat perf.rake
  require 'bundler'
  Bundler.setup

  require 'derailed_benchmarks'
  require 'derailed_benchmarks/tasks'
```

This is done so the benchmarks will be loaded before your application, this is important for some benchmarks and less for others. This also prevents you from accidentally loading these benchmarks when you don't need them.

Then you can execute your commands via rake.

To find out the tasks available you can use `$ rake -f perf.rake -T` which essentially says use the file `perf.rake` and list all the tasks.

```
$ rake -f perf.rake -T
```

## Rack Setup

Using Rails? You don't need to do anything special. If you're using Rack, you need to tell us how to boot your app. In your `perf.rake` file add a task:

```
namespace :perf do
  task :rack_load do
    DERAILED_APP = # your code here
  end
end
```

Set the constant `DERAILED_APP` to your Rack app. See [schneems/derailed_benchmarks#1](https://github.com/schneems/derailed_benchmarks/pull/1) for more info.

## Authentication

If you're trying to test an endpoint that has authentication you'll need to tell your task how to bypass that authentication. Authentication is controlled by the `DerailedBenchmarks.auth` object. There is a built in support for Devise. If you're using some other authentication method, you can write your own authentication strategy.

To enable authentication in a test run with:

```
$ USE_AUTH=true derailed exec perf:mem
```

See below how to customize authentication.

### Authentication with Devise

If you're using devise, there is a built in auth helper that will detect the presence of the devise gem and load automatically.

Create a `perf.rake` file at your root.

```
$ cat perf.rake
```

If you want you can customize the user that is logged in by setting that value in your `perf.rake` file.

```
DerailedBenchmarks.auth.user = User.find_or_create!(twitter: "schneems")
```

You will need to provide a valid user, so depending on the validations you have in your `user.rb`, you may need to provide different parameters.

If you're trying to authenticate a non-user model, you'll need to write your own custom auth strategy.

### Custom Authentication Strategy

To implement your own authentication strategy You will need to create a class that [inherits from auth_helper.rb](lib/derailed_benchmarks/auth_helper.rb). You will need to implement a `setup` and a `call` method. You can see an example of [how the devise auth helper was written](lib/derailed_benchmarks/auth_helpers/devise.rb). You can put this code in your `perf.rake` file.

```ruby
class MyCustomAuth < DerailedBenchmarks::AuthHelper
  def setup
    # initialize code here
  end

  def call(env)
    # log something in on each request
    app.call(env)
  end
end
```

The devise strategy works by enabling test mode inside of the Rack request and inserting a stub user. You'll need to duplicate that logic for your own authentication scheme if you're not using devise.

Once you have your class, you'll need to set `DerailedBenchmarks.auth` to a new instance of your class. In your `perf.rake` file add:

```ruby
DerailedBenchmarks.auth = MyCustomAuth.new
```

Now on every request that is made with the `USE_AUTH` environment variable set, the `MyCustomAuth#call` method will be invoked.

## License

MIT

## Acknowledgements

Most of the commands are wrappers around other libraries, go check them out. Also thanks to [@tenderlove](https://twitter.com/tenderlove) as I cribbed some of the Rails init code in `$ rake perf:setup` from one of his projects.

kthksbye [@schneems](https://twitter.com/schneems)

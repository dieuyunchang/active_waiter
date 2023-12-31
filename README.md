This gem is built by (choonkeat)[https://github.com/choonkeat/active_waiter] but stop updated. I cloned the gem and update to allow for ruby > 3.0

# ActiveWaiter

[![Build Status](https://travis-ci.org/choonkeat/active_waiter.svg?branch=master)](https://travis-ci.org/choonkeat/active_waiter)

A simple mechanism allowing your users to wait for the completion of your `ActiveJob`

## Scenario

You have an export PDF feature that you've implemented directly in the controller action.

As data grows, your HTTP request takes longer and starts timing out. So you decide to move the PDF generating code into a background job.

``` ruby
def index
  respond_to do |format|
    format.html
    format.pdf {
      ExportPdfJob.perform_later(@things, current_user)
      redirect_to :back, notice: "We'll email your PDF when it's done!"
    }
  end
end
```

But how do you get that PDF into the hands of your users now? Email? Push notification? Manual reload?

> You have no PDF ready for download (yet). Please reload.


## Solution

Let `ActiveWaiter` enqueue that job instead and redirect to its progress tracking page.

``` ruby
def index
  respond_to do |format|
    format.html
    format.pdf {
      uid = ActiveWaiter.enqueue(ExportPdfJob, @things, current_user)
      redirect_to active_waiter_path(id: uid)
    }
  end
end
```

``` ruby
# routes.rb
mount ActiveWaiter::Engine => "/active_waiter"
```

When the job completes, the user will be redirected to the `url` returned by the job. However, if you want the user to be presented with a download link, add `download: 1` params instead

``` ruby
redirect_to active_waiter_path(id: uid, download: 1)
```

![active_waiter mov](https://cloud.githubusercontent.com/assets/473/7785141/c4667734-01b4-11e5-8974-3a3b00b3a4b6.gif)

And we need to add a bit of code into your `ActiveJob` class

- 1) add `include ActiveWaiter::Job`
- 2) return a `url` from your `perform` method to link to the result page (or file)

``` ruby
class ExportPdfJob < ActiveJob::Base
  queue_as :default

  # (1)
  include ActiveWaiter::Job

  def perform(things, current_user)
    count = things.count.to_f
    files = []
    things.each_with_index do |thing, index|
      files << generate_pdf(thing)

      # (a)
      update_active_waiter percentage: (100 * (index+1) / count)
    end

    # (2)
    upload(combine(files)).s3_url
  rescue Exception => e

    # (b)
    update_active_waiter error: e.to_s
  end
end
```

Optionally, you can also

- a) report progress while your job runs, using `update_active_waiter(percentage:)`
- b) report if there were any errors, using `update_active_waiter(error:)`

### Configuration

By default, `ActiveWaiter` uses a simple Bootstrap layout. To use your application's layout, configure:

```ruby
ActiveWaiter.configure do |config|
  config.layout = "layouts/application"
end
```

Next, prefix any routes used in your application's layout with `main_app.`, e.g. `main_app.sign_in_path`.

This is required because `ActiveWaiter` is a Rails Engine mounted into your application,
and it doesn't know about the routes declared within your application.

#### Changing the shared cache

For deploying in production environments, `ActiveWaiter` requires a shared cache to
track the status of downloads. By default it uses the Rails cache. If you want to
use something else other than the Rails cache like redis for example, you may specify
your own implementation:

```ruby
class RedisStore
  def write(uid, value)
    $redis.set("active_waiter:#{uid}", value)
  end

  def read(uid)
    $redis.get("active_waiter:#{uid}")
  end
end

ActiveWaiter.configure do |config|
  config.store = RedisStore.new
end
```

#### Exceptions

When your job gets an exception, the error message will be written in the error message and passed along
to the user. If your job has a method `suppress_exceptions` that returns a truthy value (default false),
`ActiveWaiter::Job` will swallow the exception and not raise it - this means there will be no retry by
`ActiveJob`.

### Common Jobs

#### ActiveWaiter::EnumerableJob

If you need to wait, you're likely doing one thing slowly or many things. For the latter case, you can just
`include ActiveWaiter::EnumerableJob` and add a few interface methods

``` ruby
def before(*args); end # called once with arguments of `perform`
def enumerable; [] end # an Enumerable interface
def items_count; 1 end # called 0-n times, depending on enumerable
def foreach(item); end # called 0-n times, depending on enumerable
def after;         end # called once
def result;        end # called once
```

Here's an example from our test code, that will generate an array of range `0...count` and return the sum
of all the numbers

``` ruby
class LoopJob < ActiveJob::Base
  include ActiveWaiter::EnumerableJob

  attr_accessor :items_count, :enumerable, :result

  def before(count)
    @items_count = count
    @enumerable = count.times
    @result = 0
  end

  def foreach(item)
    @result += item
  end
end
```

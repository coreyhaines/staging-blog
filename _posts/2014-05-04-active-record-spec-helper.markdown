---
layout: post
title: Speeding Up ActiveRecord Tests
---

It is no secret that I'm a huge proponent of fast tests and their contribution to effective testing and design techniques. When doing any form of test-first development, the length of the feedback cycle when running individual tests should be as fast as possible. The design techniques associated with test-driven development (TDD), in particular, rely on taking small steps, teasing out design guidance via constant and focused refactoring. To do this, it is essential that running individual tests, the ones you are working on right now, be so fast it would be silly not to run them constantly. I'm not talking here about the time for the entire test suite, but rather for the the individual tests that you run while developing a specific component.

Of course, with Ruby on Rails, test speed has always been a huge problem. Or, rather, it isn't the individual tests that are slow to run, but rather the startup time for Rails is generally at least an order of magnitude longer than the time it takes to run the tests. This post is focused on how to test more effectively, writing tests against individual, isolated parts of your application.

<aside class='callout highlight'>
<h2>As An Aside</h2>
The recent release of Rails 4.1 includes the Spring application pre-loader to help with startup time. This is the latest in a long-line of application pre-loaders, but this one has been bundled into the core distribution. While this helps startup time, it is really just a band-aid over the real problem: loading up your entire environment when running a microtest. By loading up your entire environment, you lose one of the most important parts of test-driven development: design feedback. The feedback provided by TDD can be understood in many ways, but I like to explain it in a simple statement:

"If you find something difficult to test, change your design to make it easy to test."

If this seems a bit overly simplistic, I recommend you watch Michael Feathers great talk called *[The Deep Synergy Between Testability and Good Design](http://vimeo.com/15007792)*. In it he discusses how many instances of code smells are naturally dealt with by focusing on testability.
</aside>

I'd like to share a technique for speeding up the tests that are dependent on ActiveRecord and the database. While there is a lot of information and talk about testing the parts of your system that are independent of the Rails framework (and, frankly, most of your business logic should be isolated from Rails), there isn't a lot about how to make your tests truly effective and enjoyable when testing those parts that are integrated with Rails. This post covers the specific case where you are testing an ActiveRecord model.

But, wait, you say, aren't unit tests isolated from the database? If they touch the database, are you still writing unit tests! Perhaps, perhaps not. This post, though, is not about the meaning of the term "unit tests", or even if we should all switch to the term "[micro tests](http://anarchycreek.com/2009/05/20/theyre-called-microtests/)". It is true that almost of all of your code should be isolated from the database with one exception: code that is writing and making sql queries. That is, ActiveRecord scopes and other code that explicitly updates records in the database.

I love scopes. I find them to be a great way to emphasize "Expresses Intent" from the [4 rules of simple design](http://c2.com/cgi/wiki?XpSimplicityRules). In fact, for anything other than simple lookups using an id or a single field, I almost always use scopes. Providing a name for a lookup makes my code more understandable. It also allows me to test my business logic isolated from the details of the database. For me, TDD naturally leads to heavy scope usage.

When talking about isolation tests, we generally think of using some form of test double. Isolation, though, is about only including the parts of your system that the code needs to run. When talking about scopes, this necessarily includes the database. So, we want to test scopes with an actual connection against our database. But loading up Rails ends up taking more time than running the actual test. With a small-ish codebase, this might be a matter of only a couple seconds, but the load time increases as your application gets larger.

So, what can we do? Let's think about what it means to test a scope.

When testing against your db, you really only need to do a couple steps:

* Create a connection to the database
* Load up the code under test
* Seed the database with appropriate data
* Run the code under test
* Verify the results

This attitude, keeping isolated to just the essentials has led me to build a new spec helper called active_record_spec_helper. Here's what it looks like.

```ruby
require 'active_record'
 
connection_info = YAML.load_file("config/database.yml")["test"]
ActiveRecord::Base.establish_connection(connection_info)
 
RSpec.configure do |config|
  config.around do |example|
    ActiveRecord::Base.transaction do
      example.run
      raise ActiveRecord::Rollback
    end
  end
end
```

As an example of usage, here's a scope that we might want to test.

```ruby
class Coderetreat < ActiveRecord::Base
  def self.running_today
    where(scheduled_on: Date.today)
  end
end
```

Now, at the top of our spec file, rather than writing

```ruby
require 'spec_helper'
```

and loading the whole framework and all the dependencies, we can require our active_record_spec_helper.

```ruby
require 'active_record_spec_helper'
require 'models/coderetreat'
 
describe Coderetreat do
  describe ".running_today" do
    it "returns a coderetreat scheduled for today" do
      coderetreat = Coderetreat.create! city: "Chicago", scheduled_on: Date.today
      Coderetreat.running_today.all.should =~ [coderetreat]
    end
 
    it "does not return a coderetreat not scheduled for today" do
      coderetreat = Coderetreat.create! city: "Chicago", scheduled_on: Date.today.advance(:days => -1)
      Coderetreat.running_today.should be_empty
    end
  end
end
```

Doing a timing comparison on my machine, running with the full spec_helper looks like:

```bash
Finished in 0.04543 seconds
2 examples, 0 failures

real	0m3.004s
user	0m1.624s
sys	0m0.640s
```

Switching to using active_record_spec_helper, it looks like:

```bash
Finished in 0.04452 seconds
2 examples, 0 failures

real	0m1.224s
user	0m0.712s
sys	0m0.200s
```

That's nearly a 2-second savings for each run. This is a very small application, too. As it grows, the spec_helper version will increase its time to load. The active_record_spec_helper version, though, will stay at the 1.2 seconds. This is one of the benefits of isolated testing:

**Your tests do not change as the complexity of the rest of the application grows**.

There is another very large benefit of this: how this relates to the design feedback of TDD. Notice that we have to explicitly require the model we are testing. Because we are not loading up our entire application, we can't rely on having everything available to us; we have to be explicit about the code that is under test. Not just the code we have under test, but also its dependencies. So, if your model includes a module, you need to explicitly require that. But, why is this good? 

Rampant coupling and complex dependency graphs are one of the most significant causes of difficulty around change, and larger Rails-based applications are poster-children of these problems. While the auto-loading in Rails can be useful at times, it definitely makes it easy to hide your coupling. On the surface, this is good. Until you come back to make changes and find yourself wading through a complex game of "why is X happening, and where is that code defined."

Now, I'm not here to say don't use the auto-loader; it can be handy. Instead, pay attention to the number of dependencies you have. One way would be to list your dependencies as comments at the top of your model files. Of course, comments very rapidly rot. Instead, rely on your executable documention, your examples. By forcing yourself to load your dependencies when testing, you have a living list of dependencies. When this starts to get too large, listen to the feedback and do some refactoring. After all, TDD is about listening to the feedback your tests provide. If you mask this feedback, you lose a huge benefit.




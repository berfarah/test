---
title: Forwarding logs with Logstash
date: 2016-03-17
tags: logs, persistence
description: Centralize your data streams effectively using Logstash and Elasticsearch
---

[Logstash](https://www.elastic.co/products/logstash) is a log forwarding utility
that can help you centralize your data streams. Last week, I set up a cluster of
image derivative generators to forward their logs to Logstash. Now I have a
front-end layer on top of [Elasticsearch](https://www.elastic.co/products/elasticsearch)
that lets me query my logs quickly! There were a few things I had to figure out
along the way that I will share with you.

### Objective
We will set up Logstash to filter out relevant information from our logs, and
consolidate logs pertaining to the same item. Our input for the purpose of this
exercise will be `stdin` from the terminal, and our output will be `stdout` so
can follow along with intended outputs. We will then briefly cover how to set
Logstash up with Elasticsearch.

### Installing Logstash

##### Mac

If you're on a mac, I highly recommend using [Homebrew](http://brew.sh). If you
don't have it installed already, you can follow the instructions [their website](http://brew.sh)
to install it. Once you have it installed, run the command below to get Logstash:

```sh
brew install logstash
```

##### Linux

If you're on Linux, you may need to install Java first, and the Logstash repository next.

```sh
sudo add-apt-repository -y ppa:webupd8team/java  # Add the Oracle Java PPA to apt
sudo apt-get update                              # Update the apt package database
sudo apt-get install oracle-java8-installer      # Install Java

sudo apt-get install logstash
```

### Configuring Logstash

Once we have that, we need to define our [inputs](https://www.elastic.co/guide/en/logstash/2.2/input-plugins.html)
and [outputs](https://www.elastic.co/guide/en/logstash/2.2/output-plugins.html).
All we are doing here is forwarding our log file to stdout to see what kind of
output we will be getting.

```ruby
# logstash.conf

# For the input, we're going to be pasting into the terminal
input {
  stdin {}
}

# The output will be to stdout with a debug codec
output {
  stdout {
    codec => rubydebug
  }
}
```

Let's assume that this is what our log looks like:

```sh
# path/to/file.log

[2016-03-19T20:36:06Z] @foo.jpg: Found file!
[2016-03-19T20:36:07Z] @foo.jpg: Cutting derivative 500x500
[2016-03-19T20:36:07Z] @foo.jpg: Cutting derivative 400x400
[2016-03-19T20:36:07Z] @foo.jpg: Cutting derivative 300x300
[2016-03-19T20:36:08Z] @foo.jpg: Cutting derivative 200x200
[2016-03-19T20:36:08Z] @foo.jpg: Cutting derivative 100x100
[2016-03-19T20:36:09Z] @foo.jpg: Backing up to /bar/baz
[2016-03-19T20:36:10Z] @foo.jpg: It took 4.15s to complete!
```

Now we'll run logstash with our config file

```sh
logstash --config logstash.conf
```

Our output should look something like this:

```ruby
{
       "message" => "[2016-03-19T20:36:10Z] @foo.jpg: It took 4.15s to complete!",
      "@version" => "1",
    "@timestamp" => "2016-03-20T06:26:00.646Z",
          "host" => "battlestation.local"
}
```

##### Adding filters

Now let's start adding [filters](https://www.elastic.co/guide/en/logstash/current/filter-plugins.html)!
I'm going to start with [`grok`](https://www.elastic.co/guide/en/logstash/current/plugins-filters-grok.html) to parse out our data.
This is where that regex-foo comes in really handy!

```rb
# logstash.conf

filter {
  grok {
    # Here we are parsing out our timestamp, file and the rest of the message
    match => ["message", "\[(?<timestamp>.+)\] @(?<file>[^:]+): (?<rest>.+)"]
  }
}
```

You'll see that now you're getting the fields that you're `grok`ing parsed out:

```rb
{
       "message" => "[2016-03-19T20:36:10Z] @foo.jpg: It took 4.15s to complete!",
      "@version" => "1",
    "@timestamp" => "2016-03-20T07:12:26.709Z",
          "host" => "battlestation.local",
     "timestamp" => "2016-03-19T20:36:10Z",
          "file" => "foo.jpg",
          "rest" => "It took 4.15s to complete!"
}
```

Let's re-assign that `timestamp` to `@timestamp` using [date](https://www.elastic.co/guide/en/logstash/current/plugins-filters-date.html).

```rb
filter {
  # ...
  date {
    match => ["timestamp", "ISO8601"]
    remove_field => ["timestamp"] # We reassigned it so we don't need it anymore
  }
}
```

Now you'll find that our timestamp matches the log's timestamp, instead of the
time it was parsed.

##### Using Aggregate

To proceed, we'll have to install a plugin called [`aggregate`](https://www.elastic.co/guide/en/logstash/current/plugins-filters-aggregate.html), so let's
find the location of the Logstash binaries.

```sh
# This command should show you where the logstash binary is linked from
ls -l $(which logstash) # => /usr/local/Cellar/logstash/2.2.2/bin/logstash

# The binary should be called logstash-plugin/plugin
/usr/local/Cellar/logstash/2.2.2/bin/logstash-plugin install logstash-filter-aggregate
```

There are a few key concepts here:

* `task_id` is the unique identifier for your aggregation
* `map` is your persistent data variable across logs
* `event` is the data tied to a specific log[^1]
* `code` is how you'll be manipulating data in `map` and `event`. It takes Ruby code.

[^1]: Like `event['timestamp']` or `event['file']` in our output above

Your aggregation starting point should create the map. Subsequent aggregations
should update that map, as you want that data to persist. Finally, the last
aggregation should take that `map` and put it into the `event`.

Let's return to the example at hand. I want to start at `Found file!` and end at
`It took 4.15s to complete!`. In between, I want to keep track of all derivatives
that were made for a certain file.

```rb
filter {
  # Our starting point
  if [message] =~ "Found file!" {
    aggregate {
      task_id => "%{file}"   # We set the key to our file name
      code => ""             # This part is required, even if we leave it empty
      map_action => "create" # Our map variable is created
    }
  }

  # For all derivatives, we want to ensure that we record that their size
  if [message] =~ "Cutting" {
    grok {
      match => ["rest", "Cutting derivative (?<derivative>.+)"]
    }

    aggregate {
      task_id => "%{file}"
      # We have to manually add our derivative to the map.
      # I'm making a new array if map['derivatives'] doesn't
      # exist and adding our derivatives to it
      code => "map['derivatives'] ||= []; map['derivatives'] << event['derivative']"
      map_action => "update"
    }
  }

  # On this final message, we want to capture our time
  # and any data from previous logs
  if [message] =~ "It took" {
    grok {
      match => ["rest", "It took (?<duration>[\d\.]+)"]
    }

    aggregate {
      task_id => "%{file}"
      # Now we take all that map data and put it on the event
      # so it becomes part of this final message
      code => "event['derivatives'] = map['derivatives']"
      map_action => "update"
      end_of_task => true     # Ends the updates of the map
      timeout => 120          # It has no timeout by default
    }
  }
}
```

If you run that again via `logstash -f logstash.conf`, you should see something
like this:

```rb
{
        "message" => "[2016-03-19T20:36:10Z] @foo.jpg: It took 4.15s to complete!",
       "@version" => "1",
     "@timestamp" => "2016-03-19T20:36:10.000Z",
           "host" => "battlestation.local",
           "file" => "foo.jpg",
           "rest" => "It took 4.15s to complete!",
           "duration" => "4.15",
    "derivatives" => [
        [0] "500x500",
        [1] "400x400",
        [2] "300x300",
        [3] "200x200",
        [4] "100x100"
    ]
}
```

Fantastic, but I don't actually want all of the logs, now that this last one is
telling me everything I need. We can use the [`drop`](https://www.elastic.co/guide/en/logstash/current/plugins-filters-drop.html)
filter and stick to just this last one.

### Sending it all to Elasticsearch

One last thing. It's nice to aggregate all of this information, but it would be
even nicer if it could all be searchable in a centralized location. So let's set
up Elasticsearch and the [`elasticsearch`](https://www.elastic.co/guide/en/logstash/current/plugins-outputs-elasticsearch.html)
plugin.

For Linux, there's an official guide [here](https://www.elastic.co/guide/en/elasticsearch/reference/current/setup-repositories.html).
On OSX, you're just a `brew install elasticsearch`[^2] away!

Here's all you have
to enter to get this stuff to work with Elasticsearch:

[^2]: Isn't Homebrew great?

```rb
output {
  # ...

  elasticsearch {
    # This plugin chooses localhost:9200 by default
    # I like to override the index for each resource
    index => "derivatives"
  }
}
```

And that's it! Enjoy your setup! Thanks for reading, folks! If you'd like to see the full logstash file, you can find that [here](files/logstash.conf).

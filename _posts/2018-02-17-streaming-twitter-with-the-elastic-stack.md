---
layout: post
title: "Streaming Twitter with the Elastic Stack"
date: "2018-02-04 00:00:00 -0400"
categories: elasticsearch twitter logstash
---

## Introduction
Data visualization is an important topic in the landscape of contemporary information technology.
With the right set of tools we can identify new patterns, and solve problems we didn't
know we had.

There are a ton of buzzwords in this field. Words like _data enrichment_, _insight_, _predictive analysis_,
_intelligence discovery_, etc., frequently accompany descriptions of the technology
we're talking about. There are a lot of commonly-heard names in this space; some
of the ones I hear the most are [Hadoop](http://hadoop.apache.org/),
[MongoDB](https://www.mongodb.com/), [Cassandra](http://cassandra.apache.org/),
[Solr](http://lucene.apache.org/solr/), etc.

Like I said, I've heard about this kind of technology for years, but as a lowly
web developer I never really took the time to learn how to use any of these
databases, or the associated products that make them so powerful, even though I have
worked for several companies that use BI technology. I finally decided to change
all that and do something interesting and potentially insightful, using the
[Elastic Stack](https://www.elastic.co/products).

This post covers a lot of ground, but don't be intimidated, I was impressed
how easy it was to go from nothing to something impressive and useful, once I started.
I just happened to be learning these tools on the night of the most recent
[State of the Union Address](https://www.c-span.org/video/?439496-1/president-trump-delivers-state-union-address),
so I had the idea to create this chart you see below:

************************INSERT THE CHART******************************

It might look like I put in a lot of work, but I hardly had to lift a finger to create
that visualization. It contains counts of any tweet containing the specified keyword,
split into one-minute buckets for the time leading up to, during, and immediately
after the address.
It was satisfying, and surprisingly easy to accomplish. Ok, enough background,
let's do something neat.

## Prerequesites
We're going to need to install three of the principal parts of the Elastic Stack
for this project. They're called [Elasticsearch](https://github.com/elastic/elasticsearch),
[Kibana](https://github.com/elastic/kibana), and [Logstash](https://github.com/elastic/logstash).
Quick two second background:

* Elasticsearch: full text search engine, this is where the tweets will be stored
* Kibana: visualization engine, this is what we'll use to generate our chart
* Logstash: data ingestion tool, this is what will stream Twitter for our keywords

You are also going to need Java. I highly recommend [SDKMAN](http://sdkman.io/)
for managing Java versions as it makes it possible to switch between Java versions
easily, and you are going to need different versions for Elasticsearch/Logstash.

### Twitter app
You're also going to need a Twitter app. If you don't have one already, you can
[set one up](https://dev.twitter.com/apps/new) with your account, or create a
new account. Additionally, after you've created your app, click the
_Create my access token_ button to create the necessary oauth token.

## Setup
### Elasticsearch
First thing's first, we'll need to install Elasticsearch. You can find basic
instructions for how to download and install ES on Elastic's
[website](https://www.elastic.co/downloads/elasticsearch).

### Kibana
Likewise, you can download and install Kibana by following the instructions
[here](https://www.elastic.co/downloads/kibana).

### Logstash
Lastly, [Logstash](https://www.elastic.co/guide/en/logstash/current/installing-logstash.html).
This one is a little trickier - if you're using Java version 9, you'll need to
switch to version 8. This is where the [SDKMAN](http://sdkman.io/) tool comes
in handy, because it allows you to very easily swap between Java versions in
multiple terminal windows.

## Implementation
If you followed the guides on setting up the Elastic stack, you should now have
running instances of Elasticsearch and Kibana. There are a few ways to test your
Logstash installation, but the easiest is to supply it with a basic pipeline.

You can read more about getting started with Logstash
[here](https://www.elastic.co/guide/en/logstash/current/first-event.html), but
basically, from your Logstash install directory, run:
```
bin/logstash -e 'input { stdin {} } output { stdout {} }'
```

This command should start a logstash pipeline that moves events from the
standard input to the standard output. If you enter something into the standard
input, you should see output that looks something like this:

```
hello world
2018-02-02T04:15:04.486Z Machine-Name.local hello world
```

Ok, CTRL+C to cancel and we'll begin to configure our pipeline.

### Setting up the Twitter Input Plugin

Inside the Logstash installation directory, there's a folder named `config`.
Create a new folder in that directory called `twiter.conf`. Open it in your
editor and save this:

```
input {
  consumer_key => "{YOUR_APP_KEY}"
  consumer_secret => "{YOUR_APP_SECRET}"
  oauth_token => "{YOUR_OAUTH_TOKEN}"
  oauth_token_secret => "{YOUR_OAUTH_SECRET}"
  keywords => []
}

output {
  elasticsearch {}
}
```

All of the authentication-related information can be obtained by following the
instructions above under `Prerequesites/Twitter`.

For `keywords`, feel free to enter any words you'd like to capture tweets about.
this is an array so it should read like `keywords => ["cars", "Ford", "Chevy"]`

There are a few additional options that you may find interesting. If you include
`full_tweet => ` with `true` or `false`, it'll save the entire tweet object
returned by the Twitter API, which includes tons of data about the tweet and the
user who tweeted it. You can also tell it to `ignore_retweets => true` to skip
any retweets, `follows => []` to limit your results to the followers of given
accounts (be sure to provide user IDs instead of usernames).

There are numerous other options and customizations you can leverage; you can
read more about the Twitter input plugin [here](https://www.elastic.co/guide/en/logstash/5.5/plugins-inputs-twitter.html).

## Stash Some Tweets!
Ok, the moment has finally arrived, let's stash some tweets! If you'd like to
receive a little more feedback about the tweets you're capturing, you can add
an `stdout` output plugin to your pipeline as well like this:

```
output {
  elasticsearch {}
  stdout {}
}
```

Next, start Logstash by running `bin/logstash -f config/twitter.conf`. If all
goes well, it'll start logging Tweets that match the criteria you specified!

Let's get some visibility on what we're grabbing.

## Enter Kibana
Kibana will give you a window into your Elasticsearch instance. If you're not already
running it, go it its install directory and run `bin/kibana` or `bin\kibana.bat` if
you're on Windows. From there, open up your favorite browser and go to `http://localhost:5601`.
Kibana should load.

***************************** SHOW A PHOTO OF KIBANA *********************************

*************WRITING ON MEMORY FROM HERE - REVIEW ALL OF THIS CLOSELY********************
The first thing we're going to want to do is choose a default index to use.
Go to the management tab and choose the logstash index (by default it should be something like `logstash-2018-02-01...`).
Then open the `Discover` application, and you should see a set of documents outlined,
along with a histogram bar chart showing the volume of events processed by the stack.

Any tweet that Logstash harvested will be accounted for in this chart.
Our next step is to make some visualization of the tweets we're capturing.
Go to the visualizations app and choose new line chart. We will be creating
a histogram, and splitting our tweets into various sub-buckets.

Put them into sub-buckets and your chart visualization is done! You can save and
export this visualization to a custom dashboard, and make additional
visualizations to accompany it! And just like that, using all-free tools,
you've got access to a super useful chart!

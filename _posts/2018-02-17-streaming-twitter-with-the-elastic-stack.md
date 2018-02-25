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
[MongoDB](https://www.mongodb.com/), [Cassandra](http://cassandra.apache.org/), and
[Solr](http://lucene.apache.org/solr/).

If you're like me, you've heard about this kind of technology for years.
I never really took the time to learn how to use any of these
databases, or the associated products that make them so powerful.
I finally decided to change all that and do something interesting and
potentially insightful, using the [Elastic Stack](https://www.elastic.co/products).

This post covers a lot of ground, but don't be intimidated, I was impressed
how easy it was to go from nothing to something impressive and useful, once I started.
I just happened to be learning these tools on the night of the most recent
[State of the Union Address](https://www.c-span.org/video/?439496-1/president-trump-delivers-state-union-address),
so I had the idea to create this chart you see below:

************************INSERT THE CHART******************************

It might look like I put in a lot of work, but I hardly had to lift a finger to create
that visualization. It contains counts of any tweet containing the specified keyword,
split into one-minute buckets for the time leading up to, during, and immediately
after the address. It was satisfying, and surprisingly easy to accomplish. Ok,
enough background, let's do something neat.

## Prerequesites
We're going to need to install three of the principal parts of the Elastic Stack
for this project. They're called [Elasticsearch](https://github.com/elastic/elasticsearch),
[Kibana](https://github.com/elastic/kibana), and [Logstash](https://github.com/elastic/logstash).
Quick background:

* Elasticsearch: full text search engine, this is where the tweets will be stored
* Kibana: visualization engine, this is what we'll use to generate our chart
* Logstash: data ingestion tool, this is what will stream Twitter for our keywords

### Twitter app
You're also going to need a Twitter app. If you don't have one already, you can
[set one up](https://dev.twitter.com/apps/new) with your account, or create a
new account. Additionally, after you've created your app, click the
_Create my access token_ button to create the necessary oauth token.

## Setup
### Elasticsearch
First thing's first, we'll need to install Elasticsearch. You can find basic
instructions for how to download and install ES on Elastic's
[website](https://www.elastic.co/downloads/elasticsearch). Follow the instructions
on the download page to unzip and start Elasticsearch on your system.

The basic idea here is:

* Download the tar/zip archive
* Unzip it
* Enter the root Elastichsearch directory
* Run Elasticsearch using `bin/elasticsearch` or `bin\elasticsearch.bat` (use the `bat` file
if you're on Windows)

Once Elasticsearch has started, you should be able to see it running on the default
port of `9200` using a tool like `curl`.

### Kibana
Likewise, you can download and install Kibana by following the instructions
[here](https://www.elastic.co/downloads/kibana). Download the artifact, follow the
instructions on the page, and set Kibana to look for Elasticsearch on
`http://localhost:9200`.

Your basic steps here will be:

* Download the tar/zip archive
* Unzip it
* Enter the root Kibana directory
* Modify `config/kibana.yml` and add `elasticsearch.url: 'http://localhost:9200'`,
or whatever location your Elasticsearch is using
* Start it up with `bin/kibana` or `bin\kibana.bat` for Windows

#### Test Kibana
Before we move on, let's make sure our Kibana can talk to our Elasticsearch ok.

We're going to create an index and write an extremely simple document to it.
In your Kibana window, navigate to the `Dev Tools` tab, pictured below.

![Kibana Dev Tools]({{ "/assets/images/elk-intro/kibana-dev-tools.png" | absolute_url }})

In here, there's an app called `Console`, which lets us operate on our Elasticsearch
cluster easily. In our window, there will be a default query that looks like this:

{% highlight json linenos %}
GET _search
{
  "query": {
    "match_all": {}
  }
}
{% endhighlight %}

Delete that and replace it with:
{% highlight json linenos %}
PUT my_index/doc/1
{
  "hello": "world" 
}
{% endhighlight %}

You can execute this command by placing your insertion point in the query and
pressing `CMD+ENTER` or `CTRL+ENTER`. You should get a response that looks like:

{% highlight json linenos %}
{
  "_index": "my_index",
  "_type": "doc",
  "_id": "1",
  "_version": 1,
  "result": "created",
  "_shards": {
    "total": 2,
    "successful": 1,
    "failed": 0
  },
  "_seq_no": 0,
  "_primary_term": 1
}
{% endhighlight%}

This confirms that our Kibana is hooked up to Elasticsearch and we can
read and write to our instance using it. Nice!

### Logstash
Ok - last download. Logstash is Elastic's data ingestion tool, and it's
what we'll use to stream Twitter and harvest our Tweets.
[Download Logstash](https://www.elastic.co/downloads/logstash) and follow
the provided instructions to get it up and running. I've paraphrased
the instructions below:

* Download the tar/zip archive
* Unzip it

#### Preparing a Logstash Config
_If you already know or don't care how Logstash works at a high level, skip to the
TLDR section just below._

Logstash has a library of plugins that allow it to consume, modify, and ship
data from almost any source, _to_ almost any source. It's very useful
for this task, and you can use it as a standalone platform for all kinds of purposes.

Logstash configuration files are divided into three main sections, each of which
process _events_, or individual pieces of data, in the stream:

* *Inputs*: these plugins take data from file sources, IP ports, services, etc.
* *Filters*: plugins that allow you to mutate data mid-flight, before shipping to outputs
* *Outputs*: plugins that receive events from inputs or filters and ship them to their destination

#### TLDR Logstash Config
In the `/config` directory, create a new file called `basic.conf` and paste the code below:

```
input {
  stdin { }
}

output {
  stdout { }
}
```

Now we're ready to run Logstash! Execute this command:

`bin/logstash -f config/basic.conf`

This starts a very basic Logstash pipeline that receives your terminal's standard input,
and sends your message to the standard output. You can test this by typing a message into
your terminal and seeing its output.

```
hello world
2018-02-25T02:17:41.332Z machine.name.directory hello world
```

#### Something Less Trivial
Let's hook Logstash up to Elasticsearch.

Open your `config/basic.config` up again and change it to look like this:

```
input {
  stdin { }
}

output {
  elasticsearch { }
}
```

*Note - if your Elasticsearch instance is listening on something other than `localhost:9200`
(the default), you're going to need to specify where to send the output events. Use this
config statement instead:*

```
output {
  elasticsearch {
    hosts => ['{host ip}:{port}']
  }
}
```

Run this again with `bin/logstash -f config/basic.conf`; try shipping some messages to Elasticsearch.
Type a string into the console that's running Logstash, then press enter. Then hop back
over to Kibana and run this query:

```
GET _cat/indices
```

The output should show two indices: the `my_index` you created earlier, and another one
that looks like `logstash-*`, with the year, month, and day. For an index created on
3 Feb 2018, the index name would be `logstash-2018.02.03`.

Next run this query:

```
GET logstash-*/_search
{
  "query": {
    "match_all": {}
  }
}
```

The `*` tells Elasticsearch to match any index with a name that includes everything before
the `*` as a prefix. Under the `hits` in your result set, you should see any messages
you've sent in your Logstash terminal.

Congrats! You've got a working Elastic Stack!!

### Setting Up Our Twitter App

We're getting close! Now we're going to set up a Twitter application. This is easier
than it sounds. Do you have a Twitter account? Great! If not, take a second and
[create an account](https://help.twitter.com/en/create-twitter-account).

Go to the [Twitter Application Management](https://dev.twitter.com/apps/new) page,
click the button that says `Create New App`. Enter a `Name`, `Description`, and
`Website`. It doesn't need to be a real website, you can just use something like
`www.example.com` for that field.

When you're ready, create the application. Once you have your app, navigate to
the `Keys and Access Tokens` tab. You should see fields containing:

* `Consumer Key`
* `Consumer Secret`
* `Access Token`
* `Access Secret Token`

These are the fields you're going to need later.

### Setting up the Twitter Input Plugin

Jump back to your Logstash directory. Create a new file in `config` directory called `twiter.conf`.
Save it with the values below:

```
input {
  twitter {
    consumer_key => "{CONSUMER_KEY}"
    consumer_secret => "{CONSUMER_SECRET}"
    oauth_token => "{ACCESS_TOKEN}"
    oauth_token_secret => "{ACCESS_SECRET}"
    keywords => []
  }
}

output {
  elasticsearch {}
}
```

Let's break down what's happening in this input config. Within the `twitter` input statement,
we're specifying four security-related options. Each of these maps to one of the key values
noted in the previous section. Plug those values in.

The last option is the most interesting: `keywords`. This is where we can tell Logstash
what words are interesting to us. There are other options as well; for instance, you
could ask Logstash to show you what all the followers of a particular user are tweeting.

There are other useful options, such as ignoring retweets. You can read more about it
[in the docs](https://www.elastic.co/guide/en/logstash/current/plugins-inputs-twitter.html).

For now, put in some keywords

BEFORE WE ACTUALLY START LOGSTASH, LET'S GO AHEAD AND DELETE THE INDICES WE CREATED
JUST TO MAKE SURE WE'RE STARTING WITH A CLEAN SLATE!!

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

---
layout: post
title:  "Introduction to D3"
date:   2017-12-22 10:57:44 -0400
categories: javascript d3
---
This is the first post in a series of posts I'm writing as I learn
[D3](https://d3js.org/). The first part of the post focuses on creating a
simple bar chart.

I have a companion repo
[hosted on my GitHub](https://github.com/justinkambic/simple-d3-barchart).

I've tagged the commits in the repo to ensure there's a baseline starting
point. You can get to the beginning of each part of the series by checking out
that step. To get to the starting point of this post, run the following
[git](https://git-scm.com/) commands:

```
git clone https://github.com/justinkambic/simple-d3-barchart
cd simple-d3-barchart
git checkout step-0
```

This gives you a basic [express](https://expressjs.com/) application, with a layout like this:

```
|-- public
|   -- bar-chart.js
|   -- index.html
|-- .gitignore
|-- index.js
|-- package-lock.js
|-- package.json
```

The only files we're going to concern ourselves with right now are `public/index.html` and `public/bar-chart.js`.

## Let's start coding!

First, we need to look at a few things in our HTML.

{% highlight HTML linenos %}
<html>
  <head>
    <title>D3 Animated Bar Chart</title>
    <script src="https://d3js.org/d3.v4.min.js"></script>
    <script src="bar-chart.js" type="text/javascript"></script>
  </head>
  <body>
    Welcome to the bar chart!
  </body>
</html>
{% endhighlight %}

Most importantly here, we're including the D3 library on line 4. On the next
line, we're including our own JS file. Lastly, inside the `<body>` tag we've
got a welcome message. It's useless. Let's get rid of it.

Replace the welcome message with this:

{% highlight HTML linenos %}
  <svg class="chart"></svg>
  <script src="bar-chart.js"></script>
{% endhighlight %}

Notice we've moved the script tag from the `<head>` element down below the
`<svg>` tag. This will ensure our scripts execute after the tag is rendered.

Your updated file should look like this:

{% highlight HTML linenos %}
<html>
  <head>
    <title>D3 Animated Bar Chart</title>
    <script src="https://d3js.org/d3.v4.min.js"></script>
  </head>
  <body>
    <svg class="chart"></svg>
    <script src="bar-chart.js"></script>
  </body>
</html>
{% endhighlight %}

Now we can start writing some D3 code.

## Making the Chart
If you've never used D3, don't let that scare you. I've included links to D3's docs for each concept
we're introducing in this chapter. The quickest way to learn a new framework is by doing
something useful with it; that said, feel free to pause and go read the [docs](https://github.com/d3/d3/blob/master/API.md)
any time you want. Otherwise just follow along and we'll have a gorgeous bar chart in no time!

### Writing the Chart Code
Open up the `bar-chart.js` file. The first thing we're going to do is define a data source.
For now, we're just going to use an array we define in our code, but we'll hook up to an actual
data source later. You can define a harcoded array of `Number`s, or use the code below to get a
randomized data set every time:

{% highlight JavaScript %}
// 15 random values between 0 and 100
const data = d3.range(15).map(() => Math.round(Math.random() * 100));
{% endhighlight %}

In addition to a data source, there are a few housekeeping variables we'll set up for our purposes.
For a production chart, you'd probably want all of these to be configurable, but for the sake of the tutorial,
it'll do just fine to store them in variables at the top of the file.

{% highlight JavaScript %}
const domainMax = data => d3.max(data) * 0.05 + d3.max(data);

const
  width = 800,
  height = 500,
  barWidth = width / data.length;
{% endhighlight %}

### Scales

Next we'll define the [scales](https://github.com/d3/d3/blob/master/API.md#scales-d3-scale) for our dataset.
This tells D3 how to stretch our chart's shapes to fill up the area we've defined. We'll need separate
scales for our chart's X and Y axes.

To use scales, we will also need to define [domains](https://github.com/d3/d3-scale/blob/master/README.md#continuous_domain).
Domains tell D3 how to assign actual values to our visualizations so they use the appropriate amount of space.

#### The X-Scale
First we'll define our graph's X scale. We're going to use what's known as a [band scale](https://github.com/d3/d3-scale/blob/master/README.md#scaleBand).
Paste the following code below your data source definition:

{% highlight JavaScript %}
// domain will be 0...i, where i is the highest index in our data source
const xDomain = data.map((d, i) => i);

const xScale = d3.scaleBand()
  .range([0, width])
  .round(true)
  .padding(0.1)
  .domain(xDomain);
{% endhighlight %}

Okay, let's break down what's happening here.
* [range](https://github.com/d3/d3-scale#band_range): takes a two-element array and uses it to give the scale its upper and lower bounds.
The minimum X value in our chart will be 0, the maximum will be the value of `width` (which is 800).
* [round](https://github.com/d3/d3-scale#band_round): tells D3 to use integers for the output, rather than floating decimals.
* [padding](https://github.com/d3/d3-scale#band_padding) sets the inner and outer padding, which specifies the space between bands, and the space
next to the first and last bands respectively.
* `d3.scaleBand()`: returns a `function` that we will call when we need to get x coordinates for our graph.

Now let's make our Y scale.

#### The Y-Scale
This will be similar to the code we just wrote to get X coordinates. The big difference here is that our Y-scale is going to be a
[linear scale](https://github.com/d3/d3-scale#linear-scales) rather than a band scale.

Paste the following code into your JS file:

{% highlight JavaScript %}
const yDomain = [0, domainMax(data)];

const yScale = d3.scaleLinear()
  .domain(yDomain)
  .range([height, 0]);
{% endhighlight %}

You may have asked yourself what the `domainMax` variable was for back when we created the "housekeeping" variables above. It defines a `function`
that will return 80% of the maximum value in our data set. This will make it so that the top 20% of our chart will have whitespace, which I find appealing.
If you dislike this, you can modify the function to reduce the value by whatever amount you like.

#### Scaling the SVG
Now we're going to modify the size of our [svg](https://developer.mozilla.org/en-US/docs/Web/SVG) element. If you're not familiar
with SVG (Scalable Vector Graphics), it's essentially a 2D graphics standard intended for use with HTML. D3 is integrated with SVG, and interacting with SVG is
probably its most common use case.

Paste this code into your file:

{% highlight JavaScript %}
const svg = d3.select(".chart")
  .attr(`width, ${width}px`)
  .attr(`height, ${height}px`);
{% endhighlight %}

This code selects the first element with the class name `chart` and sets their width and height attributes to the values we defined in our "housekeeping" section.

We're also setting the variable `svg` equal to the return of [d3.select](https://github.com/d3/d3-selection#select). This will allow us to do further manipulations
to the svg element as needed.

### Where the Magic Happens
Ok, with all that setup behind us we are finally ready to get our first taste of the real power of D3! We're going to tell D3 to create a `rect` element within
our SVG for each element in our data source.

Paste this code into your file:

{% highlight JavaScript %}
svg.selectAll("rect")
  .data(data)
  .enter()
  .append("rect")
    .attr("x", (d, i) => xScale(i))
    .attr("y", d => yScale(d))
    .attr("width", xScale.bandwidth())
    .attr("height", d => height - yScale(d) + "px");
{% endhighlight %}

Whoa! What is happening here? Let's break it down line by line:
* [selectAll](https://github.com/d3/d3-selection#selection_selectAll): this function returns any elements matching the selection criteria, and adds an element
for each data point that doesn't have a match. In our case, we don't have any `rect` elements yet, so we're going to add one for each element of our data set.
* [data](https://github.com/d3/d3-selection#selection_data): allows us to specify the data we will use for the elements we're creating.
* [enter](https://github.com/d3/d3-selection#selection_enter): this will get us a collection of any nodes we've selected, and placeholders for nodes that don't have a match.
* [append](https://github.com/d3/d3-selection#selection_append): creates a new `rect` element for each element of our data set.
* [attr](https://github.com/d3/d3-selection#selection_attr): allows us to edit attributes of each element we're creating.

We're almost done, we just have one more thing to take care of.

#### Styling the Chart
We want to give our chart some spiffy styles so it will be beautiful. Create a new stylesheet file under the `public` directory called `bar-chart.js`.
Inside this file, paste this code:

{% highlight CSS %}
.chart {
    border: 1px black solid;
}

.chart rect {
    fill: steelblue;
}
{% endhighlight %}

Make sure you also reference this stylesheet in your index.html's `<head>` element. It should look like this:

{% highlight HTML %}
<link rel="stylesheet" type="text/css" href="bar-chart.css">
{% endhighlight %}

### Outputting the Chart

Ok! We're ready to serve up our file, with the included bar chart!

Have your terminal in the root of the project directory. That should be inside the `simple-d3-barchart` directory.

Run the following commands:
```
npm install
node index.js
```

`install` will tell `npm` to get any package dependencies we'll need to run our basic web app.

The second command tells `node` to run our `index.js` file, which will host our page for us!

You can access your page at `http://localhost:4000`.

You should see our chart with blue bars, and a thin black border. Congrats, you've just created your first D3 chart!

If it didn't work, it's ok, don't be discouraged. Go back and look at your HTML and JS files and compare them to what we have here:

#### index.html
{% highlight HTML linenos %}
<html>
  <head>
    <title>D3 Animated Bar Chart</title>
    <link rel="stylesheet" type="text/css" href="bar-chart.css">
    <script src="https://d3js.org/d3.v4.min.js"></script>
  </head>
  <body>
    <div>
      <svg class="chart"></svg>
      <script src="bar-chart.js" type="text/javascript"></script>
    </div>
  </body>
</html>
{% endhighlight %}

#### bar-chart.js
{% highlight JavaScript linenos %}
const data = d3.range(15).map(() => Math.round(Math.random() * 100));

const domainMax = data => d3.max(data) * 0.05 + d3.max(data);

const
  width = 800,
  height = 500,
  barWidth = width / data.length;

const xScale = d3.scaleBand()
  .range([0, width])
  .round(true)
  .padding(0.1)
  .domain(data.map((d, i) => i));

const yScale = d3.scaleLinear()
  .domain([0, domainMax(data)])
  .range([height, 0]);

const svg = d3.select(".chart")
  .attr("width", width + "px")
  .attr("height", height + "px");

svg.selectAll("rect")
  .data(data)
  .enter()
  .append("rect")
    .attr("x", (d, i) => xScale(i))
    .attr("y", d => yScale(d))
    .attr("width", xScale.bandwidth())
    .attr("height", d => height - yScale(d) + "px");
{% endhighlight %}

If you can't see what's wrong, it's okay to simply copy and paste these examples into your files. We're not practicing
typing, we're trying to learn concepts here.

## Next Steps
In the next post, we'll dig further into some of D3's charting features. We'll add axes and tooltips, animation, and a few other nifty features.
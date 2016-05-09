---
layout: post
title:  "Getting Event and Performer Data From the Ticketmaster API"
date:   2016-05-09 14:58:21 -0800
categories: ruby rails ticketmaster apis httparty
---

Poor API availability and accessibility can severely complicate or even cripple a project from day 1. I set out to include real comedian and show data in my Gigglr project knowing that I was probably facing a long journey building a series of fragile scrapers. Fortunately, I came across the wonderful Ticketmaster API with comprehensive documentation, well-organized response data, and a developer friendly sign up process.

Getting Started

Getting access to the Ticketmaster API is a breeze. Simply go to <a href="http://developer.ticketmaster.com">http://developer.ticketmaster.com</a> to create an account. You’ll need to provide some basic information like your email, app name, and website. Even if you have a tiny hobbyist project, you should instantly be approved. Ticketmaster is not tight with access like a lot of APIs.

Once you have a key, it’s a good idea to familiarize yourself with the <a href="http://developer.ticketmaster.com/products-and-docs/apis">documentation</a>(although you can just follow along here too). 

Ticketmaster has a number of APIs, but we’re going to focus on the Discovery API for getting performer and event data.

<h3>Making Attraction Requests</h3>

Ticketmaster’s AdddPI receives GET requests at the following endpoint: 

> https://app.ticketmaster.com/discovery/v2/

All API calls must have an “apikey” parameter attached.

For our first request, let’s try to get a comedian’s details. All performers, including comedians, are categorized as “Attractions” in the API so we’ll want to make calls to the endpoint for attractions: 

> https://app.ticketmaster.com/discovery/v2/attractions.json?apikey={YOUR API KEY}

Now we want to search for a specific comedian so we’ll pass a “keyword” parameter that functions like a search query: 

> https://app.ticketmaster.com/discovery/v2/attractions.json?apikey={YOUR API KEY}&keyword=Bill Burr

Here’s what the call would look like using HTTParty in Ruby:

{% highlight ruby %}

HTTParty.get(‘https://app.ticketmaster.com/discovery/v2/attractions.json?apikey={YOUR API KEY}&keyword=Bill Burr’)

{% endhighlight %}

If you get an SSL error running this, try turning off verification.

{% highlight ruby %}

HTTParty.get(‘https://app.ticketmaster.com/discovery/v2/attractions.json?apikey={YOUR API KEY}&keyword=Bill Burr’, verify: false)

{% endhighlight %}

Ticketmaster sends back the following data for this request. I’ve clipped the response a bit to focus on the real meat of the response:

{% highlight json %}
{
  "_links": {
    "self": {
      "href": "/discovery/v2/attractions.json?view=null&keyword=Bill+Burr{&page,size,sort}",
      "templated": true
    }
  },
  "_embedded": {
    "attractions": [
      {
        "name": "Bill Burr",
        "type": "attraction",
        "id": "K8vZ9175YiV",
        "test": false,
        "url": "http://ticketmaster.com/artist/961318",
        "locale": "en-us",
        "images": [
          {
            "ratio": "4_3",
            "url": "http://s1.ticketm.net/dam/a/c50/9b65a61e-2327-4293-be56-5e10f3f6ac50_12381_CUSTOM.jpg",
            "width": 305,
            "height": 225,
            "fallback": false
          },
          {
            "ratio": "3_2",
            "url": "http://s1.ticketm.net/dam/a/c50/9b65a61e-2327-4293-be56-5e10f3f6ac50_12381_TABLET_LANDSCAPE_3_2.jpg",
            "width": 1024,
            "height": 683,
            "fallback": false
          },
          ...
        ],
  ...
}
{% endhighlight %}

One of the great things about the API is they return several picture sizes for each attraction in addition to the usual information like name, id, etc.

Note that if we already know an attraction’s Ticketmaster ID, we can pass that directly. Be sure to pass an ID instead of a keyword if you can to prevent getting results for multiple attractions with the same name. Here’s what a request with Bill Burr’s ID looks like:

{% highlight ruby %}

HTTParty.get(‘https://app.ticketmaster.com/discovery/v2/attractions/K8vZ9175YiV.json?apikey={YOUR API KEY}’)

{% endhighlight %}

<h3>Event Requests</h3>

Event requests are just as simple as attraction requests, but the response data is much deeper.

A basic event request format should look like this with HTTParty:

{% highlight ruby %}

HTTParty.get(‘https://app.ticketmaster.com/discovery/v2/events.json?apikey={YOUR API KEY}’)

{% endhighlight %}

A generic call like this would give us a huge dump of event data that we don’t really want. Fortunately, we can specify if we only want events that belong to a particular attraction.

Referring back to our attraction response from earlier, we see Bill Burr’s Ticketmaster ID is “K8vZ9175YiV”. We then pass that through an attractionID parameter.

{% highlight ruby %}

HTTParty.get(‘https://app.ticketmaster.com/discovery/v2/events.json?apikey={YOUR API KEY}&attractionId=K8vZ9175YiV’)

{% endhighlight %}

There’s a lot packed into the response data so I won’t post the full response. The event data we’re interested in is buried a little bit so we’ll need to dig to get our hands on it. Here’s an example for how to get to the data, iterate over each event, and puts out each event’s information:

{% highlight ruby %}

event_response = HTTParty.get(‘https://app.ticketmaster.com/discovery/v2/events.json?apikey={YOUR API KEY}&attractionId=K8vZ9175YiV’)
event_response[“_embedded”][“events”].each do |event|
  puts “#{event[“name”]}, Time: #{event[“dates”][“start”][“localTime”]} #{event[“dates”][“start”][“localDate”]}, Venue: #{event[“_embedded”][“venues”][0][“name”]}
end

{% endhighlight %}

Notice that Ticketmaster passes us the venue information for each event. In this example, I only include the venue’s name, but you can also access the venue’s address, ID, time zone, and geographic coordinates without having to make a separate call to the venues API.

That being said, if you want to make a venues request to the API, you can easily do so.

{% highlight ruby %}

HTTParty.get(‘https://app.ticketmaster.com/discovery/v2/venues/KovZpZAE677A.json?apikey={YOUR API KEY}’)

{% endhighlight %}

Overall, I’m very impressed with the simplicity and comprehensiveness of the Ticketmaster API. They’re also generous with free API calls up to 5,000 per day.

Happy coding!
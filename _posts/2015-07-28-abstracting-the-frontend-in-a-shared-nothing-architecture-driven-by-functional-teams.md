---
layout: post
title: "A \"frontend middleware\" on top of a shared-nothing architecture"
description: "Managing the technical frontend layer on top of a vertically cut, shared-nothing architecture
is a serious challenge and keeping the idea of DRY alive and kicking among entirely separated teams
requires quite a lot of convention and discipline. Read on to learn how we approached this problem."
category: general
author: ricopfaus
tags: [javascript,coffeescript,tracking,analytics,affiliates,tagmanager,frontend]
---
{% include JB/setup %}

## The challenge
Before I explain what exactly a *"frontend middleware"* is, let me first tell you why we invented it and why it might make sense for you to use one, too. 

About one year ago, when I joined the e-commerce team at GALERIA Kaufhof working on their new webshop platform, everything was still in its early stages. The idea of having totally separated application "verticals" was the crazy new thing and the entire system was cut into loosely coupled services to keep the business logic separated. From a backend and business perspective this perfectly made sense.

> See our [Jump - Ein Technologie-Sprung bei Galeria Kaufhof](http://www.inoio.de/blog/2014/09/20/technologie-sprung-bei-galeria-kaufhof/) post (in German) for background information on the new webshop platform architecture.

But I realized very quickly that - from a frontend view - having five entirely separated functional teams working on things like integrating tracking solutions, affiliates or any other third-party content wasn't very efficient. Five independent frontend devs had to individually learn tracking APIs, implement tracking snippets, manage things like accounts within their own application's context, and so on. You should get the point. Sounded like the same job done multiple times by multiple people.

## Wait, what's the problem again?
If you are not a frontend developer you might wonder why we didn't simply embed tracking and retargeting pixels into the markup, just as the vendors tell us. Let me explain. Directly embedding third-party code may introduce two major problems: performance issues and script errors. You might know that all script tags in the markup are synchronously loaded and blocking the pageload by default. Although most vendors switched to asynchronously loading tags (using the [`async` attribute](http://caniuse.com/#feat=script-async)), synchronous loading can cause enormous performance penalties if you're not aware of the problem. Additionally there is always a risk that broken third party scripts cause Javascript errors, which - in the worst case - break and halt your entire script logic.

Besides the common performance issues and risk of errors, directly embedding third parties is also horribly inefficient and unscalable. Agreed - for one single developer, placing one global pixel in the head of the outmost template in his blog, such approach might be sufficient. But as soon as you start scaling, things become more and more unmaintainable. Imagine you split the application frontend among five or more teams and try to include 10 to 20 pixels. Each team needs to have a story (or at least some sort of task or ticket if you're not into Scrum) for adding, editing or removing every single pixel. Add project management costs on top and it quickly becomes really expensive.

Even worse - having no dedicated owner for the integration of third parties means there is nobody to ensure that the technical integration is done correctly or in any way consistent among the teams. Not to mention that, in the worst case, you need a complete application deployment (involving all verticals) to update or change a single pixel. I hope this illustrates the problem.

## The "data layer" approach
Working with tag management systems before, I got used to what I'd call the "data layer" approach pretty well. To put it simple: the website renders some kind of interesting business information (e.g. product attributes or shopping basket contents) into its page body. Often this is done using some Javascript API. The tagmanager software then takes this information and hands it over to third-party tools like analytics, affiliates or alike. A very simple example, illustrating the concept:

{% highlight javascript %}
// common example of a "data layer"
var DataLayer = window.DataLayer || [];
DataLayer.push({
  product:{
    id: 1234567890,
    name: "Book",
    price: 19.95
  }
});
{% endhighlight %}

This technique felt like a sensible solution for our architecture of rather strictly separated components. Each vertical application could independently render data into the markup, and some frontend logic would consume this data and take care of the rest (i.e. dispatching data to third parties). While thinking about it, we wanted to take it even one step further and do fancy things like declarative tracking, channel recognition and tag management within our own context.

## Meta tags to the rescue
From a technical perspective things were pretty obvious. We needed some API that the application could utilize for transporting information into the client space. Also, the implementation had to be completely generic and shouldn't force any of the structural decisions (i.e. "verticals") upon the API. An additional scripting API to call from within custom controllers would be nice, but wasn't a top priority.

Being modern frontend guys, we always avoid inlined script tags whenever possible. So we decided to not rely on Javascript for passing the data around. Instead, we are using dedicated metatags containing JSON data for handing over information from the markup to our *frontend middleware* (which we decided to call "Data Abstraction Layer", or DAL). An example metatag could look like this:

{% highlight XML %}
<!-- example for a page data object -->
<meta name="gk:dal:data" content='{
  "page":{
    "type":"homepage",
    "name":"Startseite"
  }}' />
{% endhighlight %}

This also gave us the important advantage that each vertical application could independently render any kind of data anywhere into a page's markup (even within SSI contexts) without worrying about availability or load order of a library or API. Markup reliably loads before any scripts. For asynchronously loaded content we developed another, Javascript-driven solution based on broadcasting events top-down from the DAL to its plugins. I'll talk about that later.

## Plugins, Rules and Repos
Behind the scenes and concepts the DAL is pretty lightweight and straightforward. From a technical perspective it is just an advanced plugin loader, lazy loading its plugins using *requirejs*. On initialization it collects all metatags from the page, aggregates the contained data into one large object, loads its plugins depending on supplied rules and hands over data to each of them. The more specific magic is then going on inside each plugins' code.

The plugins are standard AMD-modules returning a class with at least a constructor and a `handleEvent` method. When instantiated, the constructor receives a reference to the DAL module, the aggregated page, and an optional configuration object. The `handleEvent` method does all the magic required for handling asynchronous events happening within a page's lifecycle. A simple stub plugin could look like the following code (this time in [Coffescript](http://coffeescript.org)):

{% highlight coffeescript %}
define "gk/lib/dal/demoAffiliate", ["thirdpartylib"] (thirdpartyLib) ->
  
  # A simple demo plugin. Just provides a class container with example plugin logic
  # @implements IDALService
  class DemoAffiliate
    
    # init our third party lib
    constructor: (@dal, @data, @config)
      thirdpartyLib.init("someaccountid")
    
    # handle async events
    handleEvent: (name, data, domain) ->
      if name is "addtocart":
        # notify my third-party backend about this event
        thirdpartyLib.notify("addtocart", data.product.id)
          
{% endhighlight %}

The plugin loading is based on a simple ruleset which, optionally, executes callbacks before loading a plugin. That allows for complex rule logic, which no tagmanager could offer out-of-the-box (e.g. *if the page's type is "checkout-complete", the user is logged in, and has more than 3 articles in his basket*). This even enables us to eventually replace our external tagmanager and host the entire affiliate integration right within our own git repository. If you are a developer and/or you ever worked with external tag management GUIs (or any other, less comfortable form of affiliate integration) you might know what a great relief that is. We are even able to **fully unit test our affiliate pixels, integrated within our CD pipeline**.

## Going async
Most modern websites don't involve a new pageload for every action. Actions like e.g. opening a layer, expanding some accordion or using the off-canvas navigation may happen anytime, asynchronously, without our DAL ever being notified. For such cases we developed the `DAL.broadcast` mechanism. It offers a simple, one-way message API that allows sending event notifications directly to the DAL. Whenever a script causes an asynchronous action that should be globally broadcasted, the `DAL.broadcast` method can be called with the specific event name and an optional information object:

{% highlight javascript %}
// broadcast a client event to the DAL
DAL.broadcast("product-addtocart", {
  id: "1234567ABC",
  name: "FABIANI Jeans",
  price: 19.95
});
{% endhighlight %}

While this solved the issue of being notified about asynchronous events it introduced a new problem. We now needed to write specific controllers for any element that should fire an event. Having to write a dedicated controller for any single button actually felt quite ugly and would have caused tons of useless code. So we decided to make the tracking more declarative and introduced custom attributes that allowed to track events without additional controller logic. 

We identified three common types of client events we are usually interested in: *click/touch*, *view* and *focus*. These event types can be automatically handled and applied to the appropriate logic using the custom data-attribute syntax `data-dal-event-{type}`. This would also allow for future extensibility (thinking of *swipe*, *pinchzoom*, ...). The following examples illustrate the basic principle: 

{% highlight XML %}
<!-- simple click event without custom data -->
<button 
  name="loginLayerOpen"
  data-dal-event-click='{"name":"layer-login-open"}'
/>
{% endhighlight %}

Using this *declarative tracking* it was now easily possible to apply tracking logic to elements without touching anything but the markup. For more advanced use cases it is also possible to append additional data directly to the event data, following the same type definition and argument signature we use for the `DAL.broadcast` function itself.  

{% highlight XML %}
<!-- click event with custom data (a teaser id in this case) -->
<div 
  class="my-teaser"
  data-dal-event-click='{
    "name":"teaser-click",
    "data":{
      "teaserId":"my-fashion-teaser",
      "campaign":"some/fancy/campaign"
    }
  }' 
/>
{% endhighlight %}

## Standards and conventions FTW
At this point you might ask: *"Yeah, sounds nice, but what's all the buzz about? This doesn't look like rocket science!"* Agreed (mostly). But, that's only a small part of the cake. The majority of work went into defining conventions and standards for the various types and use cases. Which data do we have to pass to our DAL? What are the globally required fields? How do we name our pages? Which data do we need for which event and/or page? What are the more specific bits of information each vertical had to pass?

The answers to all those questions are very subjective and closely related to the field of business. Obviously, for e-commerce sites you have substantially different information you want to collect, compared to a content-driven online magazine. But in any case it boils down to some key metrics you want to collect and analyze for your business. E.g., a big part of the DAL's data is passed to our RUM (Real User Monitoring) and BI (Business Intelligence) software, which are also implemented as DAL plugins. Thus, many of our own conventions and metrics are specific to this use case. Another major part is the affiliate and basket tracking integration.

Luckily, I had done most of that long time before, when I initiated the project "tagmanager integration" for our old webshop. So I started writing down the important KPIs and metrics based on that historical data and the tracking-pixels from our tracking solution and our recommendation engine. I summed it all up in a table in our wiki, structured the table based on verticals and added a description for all keys. Then I wrote the integration stories for the vertical teams so they could integrate the right metatags and declarations in the markup.
Awesome, or so I thought.

Well, it felt awesome until I looked at the actual implementation done by the teams. The problem was that I had not been very explicit in declaring the types (and especially the formatting) for the individual metrics. I simply defined something like *on the product detail page we need a product object with the following fields: name, price, category, ...*. Even though I also defined some examples, this still led to fairly diverse implementations. Especially the price formatting was a problem, because there were multiple interpretations about how a price should be expressed (e.g.`"29,90€"` vs. `29.9` which are both valid representations of the same price value).

## Strong typing to the rescue
Since Javascript (and JSON, too) is a very loosely typed language, we needed to define abstract types (or interfaces if you're using [Typescript](http://typescriptlang.org)) that explicitly define how values have to be supplied to the DAL. This resulted in various fancy types, e.g. *DALPageData*, *DALUserData* or *DALProductData* to just name a few. Let's look at a part of the type definition for the *DALProductData* type:

{% highlight javascript %}
// @interface IDALProductData
DALProductData = {
  
  /**
   * Internal ID of this product.
   */
  "productId" : {
    "type" : "String",
    "mandatory" : true
    }
  },
  
  /**
   * EAN (International Article Number, see https://en.wikipedia.org/wiki/International_Article_Number_%28EAN%29) of this product.
   */
  "ean" : {
    "type" : "String",
    "mandatory" : true
  },
  
  /**
   * Object with price information.
   */
  "priceData": {
    "type": "DALPriceData",
    "mandatory": true,
    "apiVersion": 2
  },
  
  /* ... */

}
{% endhighlight %}

As you can see we provide a JSON structure defining the type and some other, optional fields (e.g. *mandatory*, *deprecated*, *apiVersion* and some more). This way we ensure backwards compatibility throughout the API because the teams can safely adapt to new API versions without introducing breaking changes.

When looking at the *priceData* attribute, you might notice how we used the *DALPriceData* type to solve the previously mentioned issues with the price formatting. Also, instead of just declaring price-related attributes directly within *DALProductData*, we defined our dedicated type for generic price information that we use for products, carts, orders and anything else that might come in the future. Here is an excerpt from the *DALPriceData* type:

 {% highlight javascript %}
// @interface IDALPriceData
DALPriceData = {
  
  /**
   * Current net price of the product or cart; *excluding* VAT, shipping or discount.
   */
  "net": {
    "type": "float",
    "mandatory": true,
    "apiVersion": 2
  },
  
  /**
   * VAT part of 'net' price.
   */
  "VAT": {
    "type": "float",
    "mandatory": true,
    "apiVersion": 2
  },
  
  /**
   * Total price (= net + VAT - discount); *including* VAT and *after* subtracting discount.
   */
  "total": {
    "type": "float",
    "mandatory": true,
    "apiVersion": 2
  },
  
  /* ... */

}
{% endhighlight %}

The curious reader might ask: *"Hey, what's the purpose of defining these as Javascript objects?"* Well, within the metatags the data is supplied as plain object literals. But in our test infrastructure (based on Selenium) we can now run over the entire page, take the above type definitions and compare them to the actual implementation. If there are mismatches, e.g. because a vertical didn't implement a type correctly, we raise an error and get a notification.

## Conclusion
So, finally, what is a "frontend middleware"? I introduced this term to describe a software architecture that "sits" between client and third-party space and provides a complete abstraction between those two. It replaces common tagmanagers, affiliate pixels and alike. At the same time it provides a declarative tracking API and deep integration with real-user monitoring, web analytics and other frontend-only tools commonly served by third parties (surveyforms, promolayers, onsite A/B testing, etc.). It is designed with verticalized, shared-nothing architectures, distributed over multiple functional teams in mind. Nevertheless it should also play really nice with any classic, monolithic architecture.

Thanks for your attention!

 *Wait - you actually read this far? Wow. You're either seriously bored or might be really into this stuff. Did you know **we are still looking for talented frontend devs**? [Get in touch](http://kaufhof-jobs.dvinci.de/cgi-bin/appl/selfservice.pl?action=jobdetail;job_pub_nr=F903080B-4E8E-4FBF-A284-9D1DDA8EC0DB;p=homepage) for more information.*
 
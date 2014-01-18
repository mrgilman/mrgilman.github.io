---
layout: post
title:  "Ember.js Metacast Using Ember Data 1.0"
date:   2014-01-18 11:42:30
categories: ember metacasts
---

I recently watched the [MetaCasts.tv Ember.js screencast](http://www.metacasts.tv/casts/ember-js-pts-1-4). I found it really helpful and easy to understand. My only hesitation in recommending it to others is that it's slightly out of date -- the application in the screencast used Ember 1.0 and Ember Data 0.13 and I wanted to use more recent versions. I decided to follow along with the screencast using newer versions of Ember and to take note of the differences I encountered. 

I used the `ember-rails` gem, which installed

* Ember 1.3.1
* Ember 1.0.0-beta.6
* Handlebars 1.2.1

I also used jQuery 1.10.2.

The only difference I encountered in part 1 of the screencast was with the `linkTo` helper in the templates. When I opened my console, there was a warning that `linkTo` is being deprecated in favor of `link-to`.

Part 2 of the screencast was where things started to get a little tricky. The differences between Ember 1.0 and 1.3.1 don't seem to be too great, but Ember Data has changed dramatically between 0.13 and 1.0. Many of the changes are summed up [here](https://github.com/emberjs/data/blob/master/TRANSITION.md), but I still had to read a lot of Stack Overflow posts before I could get everything working properly.

In Ember Data 1.0, `App.Country.find()` doesn't work anymore. Now we access our models by using `@store.find('country')`. But this only works in routers and contollers. The transition document notes that you can also inject the store into other components using `App.inject('component', 'store', 'store:main');` but warns that accessing the store from anywhere other than a controller or router is an antipattern. If you want to fetch models from the server in the javascript console as is demonstrated in the screencast, you can do so by typing `App.__container__.lookup('store:main').find('country')`.

Good news -- in Ember 1.0, we no longer need to configure the RESTAdapter to handle irregular plurals. It will automatically look for the `countries` and `breweries` routes instead of `countrys` and `brewerys`. \o/

When the API is namespaced, we no longer want to reopen the RESTAdapter in Ember 1.0. Instead of `DS.RESTAdaper.reopen`, we define an adapter and extend it, like so:

{% highlight coffeescript %}
  App.ApplicationAdapter = DS.RESTAdapter.extend
    namespace: "api/v1"
{% endhighlight %}


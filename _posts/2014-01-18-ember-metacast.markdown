---
layout: post
title:  "Ember.js Metacast Using Ember Data 1.0"
date:   2014-01-18 11:42:30
categories: ember
---

I recently watched the [MetaCasts.tv Ember.js screencast](http://www.metacasts.tv/casts/ember-js-pts-1-4). I found it really helpful and easy to understand. My only hesitation in recommending it to others is that it's slightly out of date -- the application in the screencast used Ember 1.0 and Ember Data 0.13 and I wanted to use more recent versions. I decided to follow along with the screencast using newer versions of Ember and to take note of the differences I encountered. 

I used the `ember-rails` gem, which installed

* Ember 1.3.1
* Ember 1.0.0-beta.6
* Handlebars 1.2.1

I also used jQuery 1.10.2.

### Part 1

I didn't run into any blockers in part 1 of the screencast. The only difference I encountered was with the `linkTo` helper in the templates. When I opened my console, there was a warning that `linkTo` is being deprecated in favor of `link-to`.

### Part 2

Part 2 of the screencast was where things started to get a little tricky. The differences between Ember 1.0 and 1.3.1 don't seem too great, but Ember Data has changed dramatically between 0.13 and 1.0. Many of the changes are summed up in the [transition document](https://github.com/emberjs/data/blob/master/TRANSITION.md), but I still had to read a lot of Stack Overflow posts before I could get everything working properly.

In Ember Data 1.0, `App.Country.find()` doesn't work anymore. Now we access our models by using `@store.find('country')`. But this only works in routers and contollers. The transition document notes that you can also inject the store into other components using `App.inject('component', 'store', 'store:main')` but warns that accessing the store from anywhere other than a controller or router is an antipattern. If you want to fetch models from the server in the javascript console as is demonstrated in the screencast, you can do so by typing `App.__container__.lookup('store:main').find('country')`.

Good news -- in Ember 1.0, we no longer need to configure the RESTAdapter to handle irregular plurals. It will automatically look for the `countries` and `breweries` routes instead of `countrys` and `brewerys`.

When the API is namespaced, we no longer want to reopen the RESTAdapter in Ember 1.0. Instead of `DS.RESTAdaper.reopen`, we extend it, like so:

{% highlight coffeescript %}
  App.Store = DS.Store.extend
    adapter: DS.RESTAdapter.extend
      namespace: "api/v1"
{% endhighlight %}

Also note that in Ember 1.0 we no longer need to call `create()` on our adapter when we define it. We also don't need to specify a revision.

### Part 3

In part 3 of the screencast, we start to work with model relationships. This is where I think Ember Data has changed the most since version 0.13.

In previous versions of Ember Data, relationship keys were expected to end in `_id` or `_ids`. Ember 1.0 will now by default look for a `breweries` key instead of `brewery_ids` in the `countries` JSON. There are a couple of ways to handle this.

The first method I tried was to change the key in the Rails `CountrySerializer` to `breweries` instead of `brewery_ids`, like so:

{% highlight ruby %}
  class CountrySerializer < ActiveModel::Serializer
    ...
    has_many :breweries, key: :breweries
  end
{% endhighlight %}

This setup worked for a while, but I ended up running into problems later on in the screencast when it came time to add new beers because Rails threw a `MassAssignmentError` when the creation params contained `country` and `brewery` instead of `country_id` and `brewery_id`.

In the end I found it was easier to modify the Ember app so that it would expect keys ending in `_id` and `_ids` for relationships. To do so, I added the following to my `app.js.coffee` file:

{% highlight coffeescript %}
  App.ApplicationSerializer = DS.RESTSerializer.extend(keyForRelationship: (rel, kind) ->
    if kind is "belongsTo"
      underscored = rel.underscore()
      underscored + "_id"
    else
      singular = rel.singularize()
      underscored = singular.underscore()
      underscored + "_ids"
  )
{% endhighlight %}

In Ember Data 1.0, the syntax for declaring `hasMany` and `belongsTo` relationships has changed slightly. Now, instead of passing "App.Brewery" to the hasMany function, we do this:

{% highlight coffeescript %}
  App.Country = DS.Model.extend
    title: DS.attr("string")
    breweries: DS.hasMany("brewery", async: true)
{% endhighlight %}

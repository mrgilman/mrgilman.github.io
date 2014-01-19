---
layout: post
title:  "Ember.js Metacast Using Ember Data 1.0"
date:   2014-01-19 15:42:30
categories: ember
---

I recently watched the [MetaCasts.tv Ember.js screencast](http://www.metacasts.tv/casts/ember-js-pts-1-4). I found it really helpful and easy to understand. One thing to note is that the screencast is slightly out of date -- the demo app uses Ember 1.0 and Ember Data 0.13 and I wanted to use more recent versions. I decided to follow along with the screencast using newer versions of Ember and to take note of the differences I encountered. Though it took me a little longer to make it through the screencast this way, I think I learned much more about how Ember works than I would have otherwise because of the additional research I had to do. If you're new to Ember I recommend that you do the same, but if you get stuck, here's what I had to do differently to get it running.

I used the `ember-rails` gem, which installed

* Ember 1.3.1
* Ember 1.0.0-beta.6
* Handlebars 1.2.1

I also used jQuery 1.10.2.

### Part 1

I didn't run into any big changes in part 1 of the screencast. The only difference I encountered was with the `linkTo` helper in the templates. When I opened my console, there was a warning that `linkTo` is being deprecated in favor of `link-to`.

### Part 2

Part 2 of the screencast was where things started to get a little tricky. The differences between Ember 1.0 and 1.3.1 don't seem too great, but Ember Data has changed dramatically between 0.13 and 1.0. Many of the changes are summed up in the [transition document](https://github.com/emberjs/data/blob/master/TRANSITION.md), but I still had to read a lot of Stack Overflow posts before I could get everything working properly.

In Ember Data 1.0, `App.Country.find()` doesn't work anymore. Now we access our models by using `@store.find('country')`. But this only works in routers and contollers. The transition document notes that you can also inject the store into other components using `App.inject('component', 'store', 'store:main')` but warns that accessing the store from anywhere other than a controller or router is an antipattern. If you want to fetch models from the server in the javascript console and trigger the API call as is demonstrated in the screencast, you can do so by typing `App.__container__.lookup('store:main').find('country')`. For examining your models, I recommend using the [Ember Inspector](https://chrome.google.com/webstore/detail/ember-inspector/bmdblncegkenkacieihfhpjfppoconhi?hl=en), which gives you some awesome Ember debugging tools in your developer console in Google Chrome.

Good news -- in Ember 1.0, we no longer need to configure the RESTAdapter to handle irregular plurals. It will automatically look for the `countries` and `breweries` routes instead of `countrys` and `brewerys`.

When the API is namespaced, we no longer want to reopen the RESTAdapter in Ember 1.0. Instead of `DS.RESTAdaper.reopen`, we extend it, like so:

{% highlight coffeescript %}
  App.Store = DS.Store.extend
    adapter: DS.RESTAdapter.extend
      namespace: "api/v1"
{% endhighlight %}

Also note that in Ember 1.0 we no longer need to call `create()` on our adapter when we define it. We also don't need to specify a revision.

### Part 3

In part 3 of the screencast, we start to work with model relationships. In previous versions of Ember Data, relationship keys were expected to end in `_id` or `_ids`. Ember 1.0 will now by default look for a `breweries` key instead of `brewery_ids` in the `countries` JSON. There are a couple of ways to handle this.

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

In Ember Data 1.0, the syntax for declaring `hasMany` and `belongsTo` relationships has changed slightly. Now, instead of passing `"App.Brewery"` to the `hasMany` function, we do this:

{% highlight coffeescript %}
  App.Country = DS.Model.extend
    title: DS.attr("string")
    breweries: DS.hasMany("brewery", async: true)
{% endhighlight %}

Likewise, in `App.Brewery`, we'll write `DS.belongsTo("country")`. I found I had to pass `async: true` to all of the `hasMany` functions so that the children would load asynchronously after the parent had already loaded.

### Part 4

Saving records to the store has changed in Ember 1.0. We no longer use the `commit()` function. When saving changes to a brewery in the `BreweryController`, we do the following:

{% highlight coffeescript %}
    brewery = @get('model')
    brewery.save()
{% endhighlight %}

Similarly, adding a new beer has changed to use `save()` instead of `commit()`. Like we did when fetching models from the server, we now call the `createRecord()` function on the `@store` instead of on the model itself. I also noticed that `country_id` would be set to null in the params and the new beer wouldn't save unless I set the `country`, not just the `country_id`, in the creation attributes. This is what my `addBeer` function looks like:

{% highlight coffeescript %}
  addBeer: (attrs) ->
    brewery = @get("model")
    country = brewery.get("country")
    beer = {
      title: attrs.title
      abv: attrs.abv
      brewery: brewery
      country: country
    }
    if !attrs.title? or attrs.title.trim() is ""
      alert "Please give your beer a name!"
    else
      beer = @store.createRecord("beer", beer)
      @get("beers").pushObject(beer)
      beer.save()
{% endhighlight %}

I noticed a deprecation warning in the console when any of the controller actions were executed, which said:

`Action handlers implemented directly on controllers are deprecated in favor of action handlers on an 'actions' object`

The solution was to wrap all of the controller actions into an `actions` object, like so:

{% highlight coffeescript %}
  actions: {
    edit: ->
      ...
    save: ->
      ...
    addBeer: (attrs) ->
      ...
  }
{% endhighlight %}

I also had to make a couple of small changes to the `Country` model in the Rails app. I modified the `before_create` to the following:

{% highlight ruby %}
  before_validation do
    if self.key.blank?
      self.key = self.title.parameterize.gsub(/-/,'')
    end
  end
{% endhighlight %}

I had to do this because Rails' `parameterize` method joins words with dashes and the `beerdb` gem validates that the key contains only lower case letters and numbers, so it was impossible to create a beer whose name was more than one word.

That's about it for the changes I had to make to get the MetaCasts.tv demo app working with Ember 1.3 and Ember Data 1.0. The screencast was definitely worth the $20 I spent on it, and I think it's especially useful if you're familiar with Rails.

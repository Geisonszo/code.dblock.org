---
layout: post
title: "Modularizing a RoR Grape API"
redirect_from: "/modularizing-a-ror-grape-api/"
date: 2011-07-23 19:17:36
tags: [rails, ruby, architecture]
comments: true
dblog_post_id: 236
---
A lot of people ask me whether we use Rails controllers for our API. We don’t, we use [Grape](https://github.com/ruby-grape/grape). Grape is a Rack-based system and a DSL that provides a cleaner separation, some API-specific  functionality and generally a better syntax. Now that we have [dealt with exceptions](/grape-trapping-all-exceptions-within-the-api) and [authentication](/grape-api-authentication-w-devise) we realized that the amount of functionality exposed in the API has grown exponentially in one single Ruby file. Let's refactor it into modules.

Here’s our current code from API v1.

{% highlight ruby %}
class Api_v1 < Grape::API
    prefix 'api'
    version 'v1'
    rescue_from :all, :backtrace => true
    error_format :json
    helpers do
        def authenticated
            if warden.authenticated?
                return true
            else
                error!('401 Unauthorized', 401)
            end
        end
    end
    namespace :me do
        # GET /api/v1/me/info
        get "info" do
            authenticated
            current_user.as_json({properties: :self})
        end
    end
end
{% endhighlight %}

We want a separate file for helpers and for the _me_ API. We can move the helpers into a module and include it normally.

{% highlight ruby %}
module ApiAuth
  def authenticated
    ...
  end
end
{% endhighlight %}

The namespace DSL is a bit tricky. Those _namespace_ and _get_ are actually namespace functions in _Grape::API_. Fortunately Ruby calls _included(module)_ for every included module. We can call the public methods on a _namespace_ function ourselves.

{% highlight ruby %}
module Api_v1_Me
  def self.included(api)
    api.namespace :me do
      # GET /api/v1/me/info
      get "info" do
        ...
      end
    end
  end
end
{% endhighlight %}

Let's combine all of this into an API class.

{% highlight ruby %}
class Api_v1 < Grape::API
  prefix 'api'
  version 'v1'
  rescue_from :all, :backtrace => true
  error_format :json
  helpers do
    include ApiAuth
  end
  include Api_v1_Me
end
{% endhighlight %}

The nice thing about this implementation is that we can now compose an API v2 with a bunch of v1 modules and some v2 ones. The not-so-nice part is the _included_ construct. I’d like to write the following.

{% highlight ruby %}
module Api_v1_Me
    include Grape::APIModule
    namespace :me do
        # GET /api/v1/me/info
        get "info" do
        ...
        end
    end
end
{% endhighlight %}

Beer to anyone who can implement this, I tried for hours, in vain - I think Grape will need some major refactoring to support modules this way. Fork Grape [on Github](https://github.com/ruby-grape/grape).

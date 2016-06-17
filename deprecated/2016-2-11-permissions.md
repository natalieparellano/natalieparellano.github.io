---
layout: post
title: A Dynamic, Database-backed Permissioning System for Rails
---

I am building an app that needs a granular set of user permissions. The app functions as a marketplace, and only certain users should be able to sell things. This is just one example. What I'd like to do is check - for any action where I might want to specify a permission - whether the active user has the permission required. 

An ex-boss (and teacher) instilled in me that all configurable options should live in the database. If I should decide one random morning that users need a new set of permissions to perform a certain action, I don't want to have to re-deploy. I should be able to change just one row in the database to enforce the new permissions. 

Below is the architecture that I came up with - welcome any suggestions for improvement!

### Checking permissions for every action

{% highlight ruby %}
# application_controller.rb

before_filter :verify_risk_allowed

# risk_helper.rb

def verify_risk_allowed
  return unless current_user # read from session

  controller = params[:controller]
  action     = params[:action]

  if current_user.allowed?( controller, action ) # the main method
    true
  elsif current_user.admin?
    true
  else
    flash[:notice] = "You don't have permission to continue. #{MSG_INVALID}"
    redirect_to root_path
  end
end
{% endhighlight %}

By setting `verify_risk_allowed` as a `before_filter`, I don't have to write code for every controller and action that might want to check permissions. I just have to remember to update the database!

### Delegating the risk assessment 

{% highlight ruby %}
# user.rb

def allowed?( controller, action )
  ActionPermission.allow?( self, controller, action )
end
{% endhighlight %}

It makes sense to call `user.allowed?`, but the logic to determine whether or not the user is allowed shouldn't live in the `User` class - that is a separate concern. For this I am using `ActionPermission`, which is the ActiveRecord-backed model that keeps track of which actions require which permission. 

### Evaluating Permissions

{% highlight ruby %}
# action_permission.rb

def self.allow?( user, controller, action )
  provider_user = user.provider_user
  allow_general?( provider_user ) && 
      allow_specific?( provider_user, controller, action )
end
{% endhighlight %}

`user.provider_user` requires a bit of explanation. When users create an account for the app, they do so with facebook (or potentially later, some other account like Google). When I enforce permissions, I'm really doing so at the level of the individual - not the account. (If someone were to say, cancel their account and then sign up again, they'd have the same app-specific user ID from facebook as they did the first time. I don't want to treat them exactly like a brand new customer.) `ProviderUser` is the model I created to hold methods and information related to the individual, vs. their account-specific actions. 

An instance of `ActionPermission` has attributes `controller`, `action`, and `accepted_codes`.

### Dividing the work

There are two method calls in `ActionPermission.allow?` - `allow_general?` and `allow_specific?`. `allow_general?` is pretty much only useful to me right now, when the app is in its very earliest stage - it is in beta testing, and only deliberately allowed users should be able to use it. So I will first check if the user is allowed to use the app at all, and then if they are allowed to do whatever specific action was requested. 

Therefore I have these private class methods: 

{% highlight ruby %}
# action_permission.rb

  # private class methods

  def allow_general?( provider_user )
    general ? general.allow?( provider_user ) : true 
  end

  def general
    find_by( controller: 'general' )
  end
{% endhighlight %}

And this public instance method:

{% highlight ruby %}
# action_permission.rb

serialize :accepted_codes, Array

def allow?( provider_user )
  if accepted_codes.present? # if this instance has any defined
    user_codes = provider_user.status_codes
    user_accepted_code = user_codes.detect{ |c| accepted_codes.include? c }
    !!( user_accepted_code )
  end
end
{% endhighlight %}

The logic is pretty straightforward - first, I only evaluate the user if the specified controller and action have `accepted_codes` defined in the database. I may place a restriction on a certain action, and then decide later that the restriction is not needed. Rather than apply the required status code to ALL users, I can simply clear `accepted_codes` on the instance.

Next, I read `status_codes` from the `provider_user`. If I can find that the user has one code in the accepted list, I go ahead and allow them to proceed.

### Putting it together

As I mentioned above, I am doing two checks - first for `ActionPermission.find_by( controller: 'general' )`, which is done for **every** action, and then a check on the specific action, which finds by the controller and action requested. This incidentally means that I can't ever create a `GeneralController`, or I might run into issues - but I don't plan on doing that. 

If the user is both generally allowed and specifically allowed, they are able to proceed - otherwise they are redirected!

### Adding user status codes

Lastly, I can't forget to add codes to my users! This is done upon account creation based on certain criteria, or I can always open the console and edit them manually. 
---
layout: post
title: Using ActiveRecord Callbacks to Sync Attributes 
---

I am currently working on a Rails app that functions as a marketplace for digital products. Each product can only be purchased once. Therefore, I'd like to ensure that whenever a purchase is completed successfully, the purchased product is always updated to reflect its "sold" status.

### Models 

There are two models at play here - 

1. `Listing` to hold details about the product for sale
+  `ListTxn` to hold details about the attempted purchase (can be successful or failed)

### Finding an Approach

My first thought was to have each `Listing` introspect on its `list_txns` to see if it can find one with a "success" status. Something like 

*listing.rb*

{% highlight ruby %}
has_many :list_txns

def status
  list_txns.detect { |lt| lt.success? } ? 'sold' : 'available'
end
{% endhighlight %}

But in this case, finding **all** the available listings would involve instantiating each `Listing`, like so 

{% highlight ruby %}
Listing.find_each { |l| l.status == 'available' } # inefficient 
{% endhighlight %}

I could avoid this with a more complicated query (which may have its own performance issues), but I don't want to bother with that just now. Instead, I'd like to store the status on the listings table, so that I can do something like 

{% highlight ruby %}
Listing.where( status: 'available' ) # better 
{% endhighlight %}

This is a bit of denormalization, but I think the benefits in clarity and performance are sufficient to justify it for now. 

### Keeping Things in Sync

I want to ensure that whenever a successful `ListTxn` is saved, the corresponding `Listing` always gets updated. 

One approach would be to handle everything in the controller:

{% highlight ruby %}
listing = Listing.find( listing_id )
# process transaction here
listing.update( status: 'sold' ) if list_txn.success?
{% endhighlight %}

This seems to be the recommended approach, but I have a few concerns - first, there are a few other scenarios where I might want to update the status of a listing (for example, refunds). I don't want to have that logic scattered everywhere. Also, if I ever need to make changes through the console, I don't want to have to remember to change two models. 

It seems logical to me that whenever an object that ***should*** impact a listing's status is created or changed, that object should force an update on its listing. This can be accomplished with callbacks.

### The Result

First step is to add an `after_commit` hook to `ListTxn`: 

*list_txn.rb*

{% highlight ruby %}
belongs_to :listing
after_commit { listing.refresh_status }
{% endhighlight %}

Then, `Listing` should know how to refresh its status:

*listing.rb*

{% highlight ruby %}
def refresh_status
  self.reload
  self[:status] = calculated_status
  self.save
end

private

  def calculated_status
    list_txns.detect { |lt| lt.success? } ? 'sold' : 'available'
  end
{% endhighlight %}

And it works!

### Takeaways

Few things I learned while working on this - 

* At first, I was using `after_save` to trigger the refresh on `Listing`. But then I started to worry, what will happen if the update on `Listing` fails to save? I don't want to trigger a rollback on `ListTxn`. `after_commit` runs *after* the changes to `ListTxn` have been committed - this is just what I want.

* However, as soon as I started using `after_commit`, my test suite began throwing a bunch of errors - it seemed that the callback was never firing! After some investigation, I found this excellent [blog post](http://www.justinweiss.com/articles/a-couple-callback-gotchas-and-a-rails-5-fix/) from Justin Weiss that describes the culprit: Rails wraps every test in its own database transaction. So `after_commit` would not be firing until after the test itself! Thankfully, there is a solution, mentioned in the post.

* Finally, you'll notice that `refresh_status` makes use of `self.reload` - why? Well, to be honest, I haven't fully figured this out - but it seems that even *after* the `ListTxn` has been committed, the associated `Listing` isn't aware that it exists:

{% highlight console %}
    53: def refresh_status
 => 54:   binding.pry
    55:   self.reload 
    56:   self[:status] = calculated_status
    57:   self.save
    58: end  

[1] pry(#<Listing>)> self.id # the current Listing
=> 2
[2] pry(#<Listing>)> ListTxn.first.listing_id # this Listing *should* have a list_txn
=> 2
[3] pry(#<Listing>)> list_txns # but it doesn't!
=> []
[4] pry(#<Listing>)> self.reload;''
=> ""
[5] pry(#<Listing>)> list_txns # now it does 
=> [#<ListTxn:0x007fd555ba8bf0
  id: 1,
  listing_id: 2
{% endhighlight %}

If anyone has the answer, please let me know! Thanks for reading. 
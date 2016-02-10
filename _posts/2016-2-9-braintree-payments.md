---
layout: post
title: Integrating Braintree Payment Processing with Rails
---

Braintree (from PayPal) is an excellent solution for processing customer payments. With Braintree, one doesn't have to worry about handling credit card or other sensitive user data, as that information is passed directly from the client to Braintree's servers. It is also pretty flexible, and well documented. Here is how I got it working with my Rails app:

### (1) Create a <a href="https://www.braintreepayments.com/get-started" target="_blank">sandbox account</a> with Braintree

API keys can be found in Account > My User > View Authorizations

### (2) Install the Braintree gem

{% highlight ruby %}
gem 'braintree', '~> 2.56.0'
{% endhighlight %}

### (3) Create config file 

*config/initializers/braintree.rb*

{% highlight ruby %}
Braintree::Configuration.environment = :sandbox
Braintree::Configuration.merchant_id = my_braintree_merchant_id
Braintree::Configuration.public_key  = my_braintree_public_key
Braintree::Configuration.private_key = my_braintree_private_key
{% endhighlight %}

### (4) Include the Braintree JavaScript SDK

In my view:

{% highlight html %}
<script src="https://js.braintreegateway.com/v2/braintree.js"></script>
{% endhighlight %}

### (5) Create a service model to handle Braintree payments processing

I decided to create a service model (not backed by ActiveRecord) to handle all Braintree functionality related to payments processing. This way, all code related to this function can live in one file, as opposed to being scattered about the app. This will make things easier if I ever decide to change processors. 

*payment_processor.rb*

{% highlight ruby %}
class PaymentProcessor
  # all relevant functionality goes here
end
{% endhighlight %}

### (6) Setup application flow

The application flow is helpfully illustrated on Braintree's site <a href="https://developers.braintreepayments.com/start/hello-server/ruby" target="_blank">here</a>.

First step is to generate a client token that is rendered in the view, as an argument to the `braintree.setup` JavaScript function provided by the SDK.

*payment_processor.rb*

{% highlight ruby %}
def self.generate_client_token
  Braintree::ClientToken.generate
end
{% endhighlight %}

*In my controller:*

{% highlight ruby %}
def new
  @client_token = PaymentProcessor.generate_client_token
end
{% endhighlight %}

*In my view:*

{% highlight html %}
<%= form_for @list_txn do |f| %>  
  <div id="dropin-container"></div>

  <!-- Other fields as required -->
  
  <div><%= f.submit %></div>  
<%end%>

<script type="text/javascript">
  braintree.setup( "<%= @client_token %>", "dropin", {
    container: "dropin-container",
    onPaymentMethodReceived: function (obj) {
      $('form#new_list_txn').append(
        "<input type='hidden' name='payment_method_nonce' value='" + obj.nonce + "'></input>"
      );      
      $( "form#new_list_txn" ).submit();
    }    
  });
</script>
{% endhighlight %}

When a user clicks "submit" on the form, the information they have provided in the Drop-in container is sent to Braintree's servers. Upon successful authentication, the "payment method <a href="https://en.wikipedia.org/wiki/Cryptographic_nonce" target="_blank">nonce</a>" is returned as an argument to the `onPaymentMethodReceived` callback. At this point, no data has been sent to the application server. 

Just a note, as per the documentation, there is really no need to define the `onPaymentMethodReceived` callback, unless you want to do additional pre-processing - appending the nonce as a hidden field and form submission are supposed to happen automatically. But this implicit behavior was not showing up for me, hence I am doing it manually. 

### (7) Create sale

Now that the `payment_method_nonce` is an input to the form, it can be parsed from the parameters to the `create` action.

*In my controller:*

{% highlight ruby %}
def create
  PaymentProcessor.create_sale( # listing and payment_method_nonce are read from params
    listing: listing, 
    options: { payment_method_nonce: payment_method_nonce }
  )  
end
{% endhighlight %}

*payment_processor.rb*

{% highlight ruby %}
def self.create_sale( merchant_account_id, list_price, options: {} )
  result = Braintree::Transaction.sale(
    :merchant_account_id  => merchant_account_id, # the secondary merchant
    :amount               => list_price, # total amount charged
    :payment_method_nonce => options[:payment_method_nonce],
    :service_fee_amount   => calculate_service_fee( list_price ), # the portion retained by the primary merchant
    :options => {
      :submit_for_settlement => true
    }
  )    

  return parse_result( result )
end

# a helper function to consistently format the result
def self.parse_result( result )
  if result.success?
    {        
      processor_id: result.transaction.id,
      status: result.transaction.status,
      details: {}
    }
  else
    {
      status: 'error',
      processor_errors: parse_errors( result ),
      details: {}
    }
  end
end
{% endhighlight %}


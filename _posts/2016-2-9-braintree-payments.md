---
layout: post
title: Integrating Braintree Payment Processing with Rails
---

Braintree (from PayPal) is an excellent solution for payment processing. With Braintree, one doesn't have to worry about handling credit card or other sensitive user data, as that information is passed directly from the client to Braintree's servers. Braintree is also pretty flexible and well documented. Here is a simple approach for getting it working with Rails.

### (1) Create a <a href="https://www.braintreepayments.com/get-started" target="_blank">sandbox account</a> with Braintree

API keys can be found in *Account > My User > View Authorizations*

### (2) Install the Braintree gem

{% highlight ruby %}
# Gemfile

gem 'braintree', '~> 2.56.0'

# config/initializers/braintree.rb

Braintree::Configuration.environment = :sandbox # or :production
Braintree::Configuration.merchant_id = my_merchant_id
Braintree::Configuration.public_key  = my_public_key
Braintree::Configuration.private_key = my_private_key
{% endhighlight %}

### (3) Include the Braintree JavaScript SDK

In the view:

{% highlight html %}
<script src="https://js.braintreegateway.com/v2/braintree.js"></script>
{% endhighlight %}

The application flow is helpfully illustrated by Braintree <a href="https://developers.braintreepayments.com/start/hello-server/ruby" target="_blank">here</a>.

### (4) Generate a client token

One approach would be to create a service model (not backed by ActiveRecord) to handle all Braintree functionality related to payments processing. This way, all code related to this function can live in one file, as opposed to being scattered about the app.

{% highlight ruby %}
# app/models/services/processors/payment_processor.rb

def self.generate_client_token
  Braintree::ClientToken.generate
end
{% endhighlight %}

### (5) Allow user to submit their payment details to Braintree

In the controller:

{% highlight ruby %}
# app/controllers/transactions_controller.rb

def new
  @transaction = Transaction.new
  @client_token = PaymentProcessor.generate_client_token
end
{% endhighlight %}

In the view:

{% highlight html %}
<%= form_for @transaction do |f| %>  
  <div id="dropin-container"></div>

  <!-- Other fields as required -->
  
  <div><%= f.submit %></div>  
<%end%>

<script type="text/javascript">
  braintree.setup( "<%= @client_token %>", "dropin", {
    container: "dropin-container",
    onPaymentMethodReceived: function (obj) {
      $('form#new_transaction').append(
        "<input type='hidden' name='payment_method_nonce' value='" + obj.nonce + "'></input>"
      );      
      $( "form#new_transaction" ).submit();
    }    
  });
</script>
{% endhighlight %}

When a user clicks "submit" on the form, the information they have provided in the drop-in container is sent to Braintree's servers. Upon successful authentication, a "payment method <a href="https://en.wikipedia.org/wiki/Cryptographic_nonce" target="_blank">nonce</a>" is returned as an argument to the `onPaymentMethodReceived` callback. At this point, no data has been sent to the application server.

Obtaining a nonce gives a "green light" for a transaction - but no sale is created until we submit the nonce back to Braintree. Having this intermediary step is great for being able to perform final checks - such as inventory control. If everything looks good, we can proceed to the next step.

Just a note - you'll notice that the `onPaymentMethodReceived` callback performs two actions - it appends the nonce as a hidden field on the form, and clicks submit on the form. As per Braintree's documentation, these steps shouldn't be necessary - they should happen automatically if no callback is defined. However I couldn't get it to work without the manual specification (if anyone can see what I'm doing wrong, please share!).

### (6) Complete the sale

Since `payment_method_nonce` is a hidden input to the form, it can be parsed from the parameters to `create` and passed back to Braintree via the API.

In the controller:

{% highlight ruby %}
# app/controllers/transactions_controller.rb

def create
  # pre-processing, such as inventory control...

  PaymentProcessor.create_sale(
    merchant_account_id,
    amount, 
    params[:payment_method_nonce]
  )
end
{% endhighlight %}

{% highlight ruby %}
# payment_processor.rb

def self.create_sale( merchant_account_id, amount, nonce )
  result = Braintree::Transaction.sale(
    :merchant_account_id  => merchant_account_id,
    :amount               => amount,
    :payment_method_nonce => nonce,
    :service_fee_amount   => calculate_service_fee( amount ), # the fee retained by the primary merchant 
    :options => {
      :submit_for_settlement => true
    }
  )    

  return parse_result( result ) # helper method to consistently format the result
end
{% endhighlight %}

In a marketplace app, there are two recipients for the transaction - the merchant (or seller) corresponding to the `merchant_account_id`, and the "master" merchant (specified in the config file above), which corresponds to the application and is used for collecting the service fee. It is important to note that `amount` is the **total** amount charged to the buyer - the service fee is deducted from the amount, and the remainder is passed to the seller. 

Note the use of `:submit_for_settlement => true` - without this option, calling `Braintree::Transaction.sale` only obtains an **authorization** for the specified amount - this confirms that (a) the payment information is valid and (b) there are sufficient funds to cover the transaction, and puts a hold on the customer's account so that they are unable to spend the funds while the authorization is valid. However funds are not actually transferred until the authorization is submitted. Rather than perform this as a separate step, you can submit for settlement right away. 

Not covered here...how to setup merchant accounts for users of the application who act as sellers. A subject for another post!
---
layout: post
title: Integrating Braintree Payment Processing with Rails
---

Braintree (from PayPal) is an excellent solution for adding payment processing to web applications. With Braintree, one doesn't have to worry about handling credit card or other sensitive user data, as that information is passed directly from the client to Braintree's servers. It is also pretty flexible and well documented. Here is a simple approach for getting it working with Rails.

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

One approach would be to create a service model (not backed by ActiveRecord) to handle all Braintree functionality.

{% highlight ruby %}
# app/models/services/processors/payment_processor.rb

def self.generate_client_token
  Braintree::ClientToken.generate
end
{% endhighlight %}

In the controller:

{% highlight ruby %}
# app/controllers/transactions_controller.rb

def new
  @transaction = Transaction.new
  @client_token = PaymentProcessor.generate_client_token
end
{% endhighlight %}

### (5) Allow user to submit their payment details to Braintree

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

What does this do? When a user views the page, embedded within the form will be a "drop-in" container with inputs for payment details - in the case of my application, credit card information. When the user clicks the "submit" button, the information they have provided in the drop-in container is sent to Braintree's servers, where Braintree will determine if the information is valid (such as by consulting with the credit card networks - or, if paying by PayPal, by validating the user's login credentials). 

If everything checks out, Braintree returns a "payment method <a href="https://en.wikipedia.org/wiki/Cryptographic_nonce" target="_blank">nonce</a>" as an argument to the `onPaymentMethodReceived` callback. 

This is like getting a "green light" for a transaction - but a transaction isn't actually initiated until we submit the nonce back to Braintree along with essential details such as how much we want to charge the user. 

Just a note - you'll notice that the `onPaymentMethodReceived` callback performs two actions - it appends the nonce as a hidden input to the form, and submits the form (finally) to the application's server. As per Braintree's documentation, these things should happen automatically if no callback is defined. However I couldn't get it to work without the manual specification (if anyone can see what I'm doing wrong, please share!).

### (6) Complete the sale

Since `payment_method_nonce` is a hidden input to the form, it can be parsed from the parameters to `create`.

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
    :service_fee_amount   => calculate_service_fee( amount ), # the fee retained by the master merchant 
    :options => {
      :submit_for_settlement => true
    }
  )    

  return parse_result( result ) # helper method to consistently format the result
end
{% endhighlight %}

In a marketplace app, there are two parties getting paid - the seller, and the marketplace (the application itself) which receives a "service fee" or cut of the transaction. In `PaymentProcessor.create_sale`, the `merchant_account_id` corresponds to the merchant account for the seller. The marketplace is identified by the `merchant_id` in the config file. It is important to note that `amount` is the **total** amount charged to the buyer - the service fee is deducted from the amount, and the remainder is passed to the seller. 

Note the use of `:submit_for_settlement => true` - without this option, calling `Braintree::Transaction.sale` only obtains an **authorization** to charge the specified `amount` to the buyer - an authorization confirms that (a) the payment information is valid and (b) there are sufficient funds in the buyer's account to cover the transaction. It also puts a hold on the buyer's account so that they cannot spend the funds while the authorization is valid. However no money is transferred until the authorization is submitted. Rather than perform this as a separate step, you can submit for settlement right away. 

Not covered here...how to setup merchant accounts for users of the application who act as sellers. A subject for another post!
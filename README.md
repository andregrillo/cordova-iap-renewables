# Cordova Purchase Plugin

> In-App Purchases for Cordova

## Summary

This plugin allows **In-App Purchases** for iOS to be made from **Cordova** applications.

## Installation

### Install the plugin (Cordova)

```sh
cordova plugin add https://github.com/andregrillo/cordova-iap-renewables.git
```

### Install recommended plugins

<details>
<summary>
Install <strong>cordova-plugin-network-information</strong> (click for details).
</summary>


Sometimes, the plugin cannot connect to the app store because it has no network connection. It will then retry either:

* periodically after a certain amount of time;
* when the device fires an ['online'](https://developer.mozilla.org/en-US/docs/Web/Events/online) event.

The [cordova-plugin-network-information](https://github.com/apache/cordova-plugin-network-information) plugin is required in order for the `'online'` event to be properly received in the Cordova application. Without it, this plugin will only be able to use the periodic check to determine if the device is back online.

</details>

### Setup

This plugin is mostly events based.
You will need to register listeners to changes happening to the products
you register.

The core of the listening mechanism is the [`when()`] method. It allows you to
be notified of changes to one or a set of products using a [`query`] mechanism:
```js
    store.when("product").updated(refreshScreen);
    store.when("subscription").owned(unlockApp);
    etc.
```
The `updated` event is fired whenever one of the fields of a product is
changed (its `owned` status for instance).

This event provides a generic way to track the statuses of your purchases,
to unlock features when needed and to refresh your views accordingly.

### Registering products 

The store needs to know the type and identifiers of your products before you
can use them in your app.

Use [`store.register()`] before your first call to
[`store.refresh()`].

It's an array with all products to be used ("Alias" is optional).
The product `id` and `type` must match the products previously defined in Apple website.
```
store.register([
	  {
        id:    'com.outsystemsenterprise.biolab.destress.dev.12monthsAutoRenew',
        alias: "12months-AutoRenew",
        type:   store.PAID_SUBSCRIPTION  },
      {
      	id:    'com.outsystemsenterprise.biolab.destress.dev.1monthAutoRenew',
        alias: "1month-AutoRenew",
        type:   store.PAID_SUBSCRIPTION  },
    ]);
```
Once registered, you can use [`store.get()`] to retrieve
the [`product object`] from the store (more info about this below).


## Checking Store if it's ready

```
store.ready(function() {
        console.log("\\o/ STORE READY \\o/");
    }); 

```

## Refresh Store

After we've done our setup, we tell the store to do it's first refresh. 
Nothing will happen if we do not call the refresh method.

This will initiate all the complex behind-the-scene work, to load product
data from the servers and restore whatever already have been
purchased by the user.

Note that you can call this method again later during the application
execution to re-trigger all that hard-work. It's kind of expensive in term of
processing, so you'd better consider it twice.

One good way of doing it is to add a "Refresh Purchases" button in your
applications settings. This way, if delivery of a purchase failed or
if a user wants to restore purchases he made from another device, he'll
have a way to do just that.

_NOTE:_ It is a required by the Apple AppStore that a "Refresh Purchases"
        button be visible in the UI.

```
store.refresh();
```

## <a name="update"></a> *store.update()*

Refresh the historical state of purchases and price of items.
This is required to know if a user is eligible for promotions like introductory
offers or subscription discount.

It is recommended to call this method right before entering your in-app
purchases or subscriptions page.

You can of `update()` as a light version of `refresh()` that won't ask for the
user password. Note that this method is called automatically for you on a few
useful occasions, like when a subscription expires.


### Displaying products

Right after you registered your products, nothing much is known about them
except their `id`, `type` and an optional `alias`.

When you perform the initial [`refresh()`](#refresh) call, the store's server will
be contacted to load informations about the registered products: human
readable `title` and `description`, `price`, etc.

This isn't an optional step. Apple requires you
to display information about a product as retrieved from their server: no
hard-coding of price and title allowed! This is also convenient for you
as you can change the price of your items knowing that it'll be reflected instantly
on your clients' devices.

However, the information may not be available when the first view that needs
them appears on screen. The best option is to have your view monitor
changes made to the product.


### Monitor Changes

Let's demonstrate this with an example:

```js
    // method called when the screen showing your purchase is made visible
    function process() {
        example();
        store.when("com.outsystemsenterprise.biolab.destress.dev.12monthsAutoRenew").updated(example());
    }

    function example() {

        // Get the product from the pool.
        var product = store.get("com.outsystemsenterprise.biolab.destress.dev.12monthsAutoRenew");

        if (!product) {
        	//Product not found or not registered.
        }
        else if (product.state === store.REGISTERED) {
            //Product registered
        }
        else if (product.state === store.INVALID) {
        	//Invalid Product State
        }
        else {
            // Good! Product loaded and valid.

            // Is this product owned? 
            if (product.owned)
            	//Ok, the user owns the product.
            else
                //The user does not own the product

            // Is an order for this product in progress? Can't be ordered right now?
            if (product.canPurchase)
                //The user can purchase
            else
                //The user can't purchase
        }
    }

    // method called when the view is hidden
    function hide() {
        // stop monitoring the product
        store.off();
    }
```

In this code, the function `example` will manipulate the purchase element whatever
happens to the product. When the view is hidden, we stop listening to changes
(`store.off(example)`).


### <a name="purchasing"></a> Purchasing

#### Initiate a purchase

Purchases are initiated using the [`store.order(product)`](#order) method.

The store will manage the internal purchase flow that'll end:

 - with an `approved` [event](#events). The product enters the `APPROVED` state.
 - with a `cancelled` [event](#events). The product gets back to the `VALID` state.
 - with an `error` [event](#events). The product gets back to the `VALID` state.

See [product life-cycle](#life-cycle) for details about product states.

#### finish a purchase

Once the transaction is approved, the product still isn't owned: the store needs
confirmation that the purchase was delivered before closing the transaction.

To confirm delivery, you'll use the [`product.finish()`](#finish) method.

#### example usage

During initialization:
```js
store.when("12months-AutoRenew").approved(function(product) {
        product.finish();
});
```

When the purchase button is clicked:
```js
store.order("12months-AutoRenew");
```
The code right above will start the purchase process and will trigger the "when approved" lines mentioned before, then finishing (confirming) the purchase.

#### un-finished purchases

If your app wasn't able to deliver the content, `product.finish()` won't be called.

The `approved` event will be re-triggered the next time you
call [`store.refresh()`](#refresh), which can very well be the next time
the application starts. Pending transactions are persistant.

#### simple case

In the most simple case, you may just want to finish all purchases automatically. You can do it this way:
```js
store.when("12months-AutoRenew").approved(function(product) {
    product.finish();
});
```

NOTE: the "product" query will match any purchases (see [here](#queries) to learn more details about queries).

### Subscriptions

Typically, you'll enable and disable access to your content this way:
```js
store.when("12months-AutoRenew").updated(function(product) {
    if (product.owned)
    	//User owns the product
    else
        //User does not own the product
});
```

# <a name="store"></a>*store* object ##

`store` is the global object exported by the purchase plugin.

As with any other plugin, this object shouldn't be used before
the "deviceready" event is fired. Check cordova's documentation
for more details if needed.

Find below all public attributes and methods you can use.

## <a name="verbosity"></a>*store.verbosity*

The `verbosity` property defines how much you want `store.js` to write on the console. Set to:

 - `store.QUIET` or `0` to disable all logging (default)
 - `store.ERROR` or `1` to show only error messages
 - `store.WARNING` or `2` to show warnings and errors
 - `store.INFO` or `3` to also show information messages
 - `store.DEBUG` or `4` to enable internal debugging messages.

See the [logging levels](#logging-levels) constants.

## Constants

### product types

    store.FREE_SUBSCRIPTION         = "free subscription";
    store.PAID_SUBSCRIPTION         = "paid subscription";
    store.NON_RENEWING_SUBSCRIPTION = "non renewing subscription";
    store.CONSUMABLE                = "consumable";
    store.NON_CONSUMABLE            = "non consumable";

### error codes

    store.ERR_SETUP               = ERROR_CODES_BASE + 1; //
    store.ERR_LOAD                = ERROR_CODES_BASE + 2; //
    store.ERR_PURCHASE            = ERROR_CODES_BASE + 3; //
    store.ERR_LOAD_RECEIPTS       = ERROR_CODES_BASE + 4;
    store.ERR_CLIENT_INVALID      = ERROR_CODES_BASE + 5;
    store.ERR_PAYMENT_CANCELLED   = ERROR_CODES_BASE + 6; // Purchase has been cancelled by user.
    store.ERR_PAYMENT_INVALID     = ERROR_CODES_BASE + 7; // Something suspicious about a purchase.
    store.ERR_PAYMENT_NOT_ALLOWED = ERROR_CODES_BASE + 8;
    store.ERR_UNKNOWN             = ERROR_CODES_BASE + 10; //
    store.ERR_REFRESH_RECEIPTS    = ERROR_CODES_BASE + 11;
    store.ERR_INVALID_PRODUCT_ID  = ERROR_CODES_BASE + 12; //
    store.ERR_FINISH              = ERROR_CODES_BASE + 13;
    store.ERR_COMMUNICATION       = ERROR_CODES_BASE + 14; // Error while communicating with the server.
    store.ERR_SUBSCRIPTIONS_NOT_AVAILABLE = ERROR_CODES_BASE + 15; // Subscriptions are not available.
    store.ERR_MISSING_TOKEN       = ERROR_CODES_BASE + 16; // Purchase information is missing token.
    store.ERR_VERIFICATION_FAILED = ERROR_CODES_BASE + 17; // Verification of store data failed.
    store.ERR_BAD_RESPONSE        = ERROR_CODES_BASE + 18; // Verification of store data failed.
    store.ERR_REFRESH             = ERROR_CODES_BASE + 19; // Failed to refresh the store.
    store.ERR_PAYMENT_EXPIRED     = ERROR_CODES_BASE + 20;
    store.ERR_DOWNLOAD            = ERROR_CODES_BASE + 21;
    store.ERR_SUBSCRIPTION_UPDATE_NOT_AVAILABLE = ERROR_CODES_BASE + 22;
    store.ERR_PRODUCT_NOT_AVAILABLE = ERROR_CODES_BASE + 23; // Error code indicating that the requested product is not available in the store.
    store.ERR_CLOUD_SERVICE_PERMISSION_DENIED = ERROR_CODES_BASE + 24; // Error code indicating that the user has not allowed access to Cloud service information.
    store.ERR_CLOUD_SERVICE_NETWORK_CONNECTION_FAILED = ERROR_CODES_BASE + 25; // Error code indicating that the device could not connect to the network.
    store.ERR_CLOUD_SERVICE_REVOKED = ERROR_CODES_BASE + 26; // Error code indicating that the user has revoked permission to use this cloud service.
    store.ERR_PRIVACY_ACKNOWLEDGEMENT_REQUIRED = ERROR_CODES_BASE + 27; // Error code indicating that the user has not yet acknowledged Apple’s privacy policy for Apple Music.
    store.ERR_UNAUTHORIZED_REQUEST_DATA = ERROR_CODES_BASE + 28; // Error code indicating that the app is attempting to use a property for which it does not have the required entitlement.
    store.ERR_INVALID_OFFER_IDENTIFIER = ERROR_CODES_BASE + 29; // Error code indicating that the offer identifier is invalid.
    store.ERR_INVALID_OFFER_PRICE = ERROR_CODES_BASE + 30; // Error code indicating that the price you specified in App Store Connect is no longer valid.
    store.ERR_INVALID_SIGNATURE = ERROR_CODES_BASE + 31; // Error code indicating that the signature in a payment discount is not valid.
    store.ERR_MISSING_OFFER_PARAMS = ERROR_CODES_BASE + 32; // Error code indicating that parameters are missing in a payment discount.

### product states

    store.REGISTERED = 'registered';
    store.INVALID    = 'invalid';
    store.VALID      = 'valid';
    store.REQUESTED  = 'requested';
    store.INITIATED  = 'initiated';
    store.APPROVED   = 'approved';
    store.FINISHED   = 'finished';
    store.OWNED      = 'owned';
    store.DOWNLOADING = 'downloading';
    store.DOWNLOADED = 'downloaded';

### logging levels

    store.QUIET   = 0;
    store.ERROR   = 1;
    store.WARNING = 2;
    store.INFO    = 3;
    store.DEBUG   = 4;

### validation error codes

    store.INVALID_PAYLOAD   = 6778001;
    store.CONNECTION_FAILED = 6778002;
    store.PURCHASE_EXPIRED  = 6778003;
    store.PURCHASE_CONSUMED = 6778004;
    store.INTERNAL_ERROR    = 6778005;
    store.NEED_MORE_DATA    = 6778006;

### special purpose

    store.APPLICATION = "application";

 ## <a name="product"></a>*store.Product* object ##

Most events methods give you access to a `product` object.

Products object have the following fields and methods.

### *store.Product* public attributes

 - `product.id` - Identifier of the product on the store
 - `product.alias` - Alias that can be used for more explicit [queries](#queries)
 - `product.type` - Family of product, should be one of the defined [product types](#product-types).
 - `product.group` - Name of the group your subscription product is a member of (default to `"default"`). If you don't set anything, all subscription will be members of the same group.
 - `product.state` - Current state the product is in (see [life-cycle](#life-cycle) below). Should be one of the defined [product states](#product-states)
 - `product.title` - Localized name or short description
 - `product.description` - Localized longer description
 - `product.priceMicros` - Price in micro-units (divide by 1000000 to get numeric price)
 - `product.price` - Localized price, with currency symbol
 - `product.currency` - Currency code (optionaly)
 - `product.countryCode` - Country code. Available only on iOS
 - `product.loaded` - Product has been loaded from server, however it can still be either `valid` or not
 - `product.valid` - Product has been loaded and is a valid product
   - when product definitions can't be loaded from the store, you should display instead a warning like: "You cannot make purchases at this stage. Try again in a moment. Make sure you didn't enable In-App-Purchases restrictions on your phone."
 - `product.canPurchase` - Product is in a state where it can be purchased
 - `product.owned` - Product is owned
 - `product.deferred` - Purchase has been initiated but is waiting for external action (for example, Ask to Buy on iOS)
 - `product.introPrice` - Localized introductory price, with currency symbol
 - `product.introPriceMicros` - Introductory price in micro-units (divide by 1000000 to get numeric price)
 - `product.introPricePeriod` - Duration the introductory price is available (in period-unit)
 - `product.introPricePeriodUnit` - Period for the introductory price ("Day", "Week", "Month" or "Year")
 - `product.introPricePaymentMode` - Payment mode for the introductory price ("PayAsYouGo", "UpFront", or "FreeTrial")
 - `product.ineligibleForIntroPrice` - True when a trial or introductory price has been applied to a subscription. Only available after [receipt validation](#validator). Available only on iOS
- `product.discounts` - Array of discounts available for the product. Each discount exposes the following fields:
   - `id` - The discount identifier
   - `price` - Localized price, with currency symbol
   - `priceMicros` - Price in micro-units (divide by 1000000 to get numeric price)
   - `period` - Number of subscription periods
   - `periodUnit` - Unit of the subcription period ("Day", "Week", "Month" or "Year")
   - `paymentMode` - "PayAsYouGo", "UpFront", or "FreeTrial"
   - `eligible` - True if the user is deemed eligible for this discount by the platform
 - `product.downloading` - Product is downloading non-consumable content
 - `product.downloaded` - Non-consumable content has been successfully downloaded for this product
 - `product.additionalData` - additional data possibly required for product purchase
 - `product.transaction` - Latest transaction data for this product (see [transactions](#transactions)).
 - `product.expiryDate` - Latest known expiry date for a subscription (a javascript Date)
 - `product.lastRenewalDate` - Latest date a subscription was renewed (a javascript Date)
 - `product.billingPeriod` - Duration of the billing period for a subscription, in the units specified by the `billingPeriodUnit` property. (_not available on iOS < 11.2_)
 - `product.billingPeriodUnit` - Units of the billing period for a subscription. Possible values: Minute, Hour, Day, Week, Month, Year. (_not available on iOS < 11.2_)


### *store.Product* public methods

#### <a name="finish"></a>`product.finish()` ##

Call `product.finish()` to confirm to the store that an approved order has been delivered.
This will change the product state from `APPROVED` to `FINISHED` (see [life-cycle](#life-cycle)).

As long as you keep the product in state `APPROVED`:

 - the money may not be in your account (i.e. user isn't charged)
 - you will receive the `approved` event each time the application starts,
   where you should try again to finish the pending transaction.

##### example use
```js
store.when("12months-AutoRenew").approved(function(product){
    // synchronous
    app.unlockFeature();
    product.finish();
});
```

```js
store.when("12months-AutoRenew").approved(function(product){
    // asynchronous
    app.downloadFeature(function() {
        product.finish();
    });
});
```

### life-cycle

A product will change state during the application execution.

Find below a diagram of the different states a product can pass by.

    REGISTERED +--> INVALID
               |
               +--> VALID +--> REQUESTED +--> INITIATED +-+
                                                          |
                    ^      +------------------------------+
                    |      |
                    |      |            
                    |      |             
                    |      +--> APPROVED +--------------------------------+--> FINISHED +--> OWNED
                    |                                                             |
                    +-------------------------------------------------------------+


#### states definitions

 - `REGISTERED`: right after being declared to the store using [`store.register()`](#register)
 - `INVALID`: the server didn't recognize this product, it cannot be used.
 - `VALID`: the server sent extra information about the product (`title`, `price` and such).
 - `REQUESTED`: order (purchase) requested by the user
 - `INITIATED`: order transmitted to the server
 - `APPROVED`: purchase approved by server
 - `FINISHED`: purchase delivered by the app (see [Finish a Purchase](#finish-a-purchase))
 - `OWNED`: purchase is owned (only for subscriptions)

#### state changes

Each time the product changes state, the appropriate event is triggered.

Learn more about events [here](#events) and about listening to events [here](#when).


## <a name="errors"></a>*store.Error* object

All error callbacks takes an `error` object as parameter.

Errors have the following fields:

 - `error.code` - An integer [error code](#error-codes). See the [error codes](#error-codes) section for more details.
 - `error.message` - Human readable message string, useful for debugging.

## <a name="error"></a>*store.error(callback)*

Register an error handler.

`callback` is a function taking an [error](#errors) as argument.

### example use:

    store.error(function(e){
        console.log("ERROR " + e.code + ": " + e.message);
    });



### Reserved keywords
Some reserved keywords can't be used in the product `id` and `alias`:

 - `product`
 - `order`
 - `registered`
 - `valid`
 - `invalid`
 - `requested`
 - `initiated`
 - `approved`
 - `owned`
 - `finished`
 - `downloading`
 - `downloaded`
 - `refreshed`


#### Apple App Store specific data

Refer to [this documentation for iOS](https://developer.apple.com/library/ios/releasenotes/General/ValidateAppStoreReceipt/Chapters/ReceiptFields.html#//apple_ref/doc/uid/TP40010573-CH106-SW1).

**Transaction Fields (Subscription)**

```
    appStoreReceipt:"appStoreReceiptString"
    id : "idString"
    original_transaction_id:"transactionIdString",
    "type": "ios-appstore"
```

##### restore purchases example usage

Add a "Refresh Purchases" button to call the `store.refresh()` method, like:

```html
<button onclick="restorePurchases()">Restore Purchases</button>
```

```js
function restorePurchases() {
   showProgress();
   store.refresh().finished(hideProgress);
}
```

To make the restore purchases work as expected, please make sure that
the "approved" event listener had be registered properly
and, in the callback, `product.finish()` is called after handling.


## <a name="manageSubscriptions"></a>*store.manageSubscriptions()*

Opens the Manage Subscription page (in iOS Settings app),
where the user can change his/her subscription settings or unsubscribe.

##### example usage

```js
   store.manageSubscriptions();
```


## <a name="manageBilling"></a>*store.manageBilling()*

Opens the Manage Billing page (in iOS Settings app),
where the user can update his/her payment methods.

##### example usage

```js
   store.manageBilling();
```


## <a name="redeem"></a>*store.redeem()*

Redeems a promotional offer from within the app.

* On iOS, calling `store.redeem()` will open the Code Redemption Sheet.
  * See the [offer codes documentation](https://developer.apple.com/app-store/subscriptions/#offer-codes) for details.


##### example usage

```js
   store.redeem();
```

## *store.log* object
### `store.log.error(message)`
Logs an error message, only if `store.verbosity` >= store.ERROR
### `store.log.warn(message)`
Logs a warning message, only if `store.verbosity` >= store.WARNING
### `store.log.info(message)`
Logs an info message, only if `store.verbosity` >= store.INFO
### `store.log.debug(message)`
Logs a debug message, only if `store.verbosity` >= store.DEBUG


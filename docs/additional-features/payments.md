---
title: Payments
---

- [Installation](#installation)
- [Available Payment Gateway](#available-payment-gateways)
- [How to charge users](#how-to-charge-users)
- [Manage Payment Accounts](#manage-payment-accounts)
- [Payment Transactions](#payment-transactions)
- [Developing own Payment Gateway Containers](#developing-own-payment-gateway-containers)
- [Mocking the real payment call for Testing](#mocking-the-real-payment-call-for-testing)

## Installation {#installation}
```shell
composer require apiato/payment-container
```
Now run `php artisan migrate`

## Supported Payment Gateways {#available-payment-gateways}

Currently, Apiato Supports the following payment gateways:

* `Stripe`

If your payment gateway is not supported, build it and contribute your container back.

**For any supported gateway you want to use, set Tokens and Secrets in the `.env`**
```
STRIPE_KEY=
STRIPE_SECRET=
```
## How to charge users {#how-to-charge-users}

1. On the Model you want to charge:
  1. Use the `App\Containers\Payment\Traits\ChargeableTrait`
  1. add the `\App\Containers\Payment\Contracts\ChargeableInterface` Interface
  1. Then add this relation
   ```
   public function paymentAccounts(): HasMany
   {
        return $this->hasMany(PaymentAccount::class);
   }
   ```

2) To charge a user, the user must first create a payment account (Stripe, PayPal, WePay,...). Use the respective
   endpoints to create those endpoints (`createStripeAccount`, ...). A User may have multiple
   `PaymentAccount`.

3) Then charge the user like this `$user->charge($account, $amount);` by providing the `$account` and the `$amount`.

You can get `$account` from the user this way `$user->paymentAccounts`. This will return a Collection of all the
user payment accounts to select one.

Example:

```php
// in this example we are selecting a random payment account.
$acccount = $user->paymentAccounts->first();

$amount = 9.99;

// charge the user with the 'USD' currency
$user->charge($acccount, $amount, 'USD');
```

`$user->paymentAccounts` will return a *generic* `PaymentAccount` which may be transformed to the dedicated
payment account (`StripeAccount`, `PaypalAccount`, ...). You can call the `accountable()` function on the selected
payment in order to do this. For more information about this, we refer to the
[Polymorphic Relationships](https://laravel.com/docs/eloquent-relationships#polymorphic-relations) in the
official Laravel Docs.

## Manage Payment Accounts {#manage-payment-accounts}

Payment contains provides some generic routes in order to allow users to manage their own `PaymentAccount`:

- `GET /user/paymentaccounts` : Get all available `PaymentAccount`s for the current `User`.
- `GET /user/paymentaccounts/{id}`: Get the details of one specific `PaymentAccount`.
- `PATCH /user/paymentaccounts/{id}`: Update a `PaymentAccount` (this route does **not** update the credentials for the
  corresponding payment gateway).
- `DELETE /user/paymentaccounts/{id}`: Delete a `PaymentAccount` including the payment gateway details (e.g., user
  credentials for `Stripe`).
- To create a payment account use the dedicated endpoint (`createStripeAccount`, `createWepayAccount`,...) provided by
  the payment gateway container (Stripe, WePay,...). Each payment container has its own endpoint to `create` and
  `update` account details, since each payment requires different data.

## Payment Transactions {#payment-transactions}

When charging the user with a specific gateway (e.g., Paypal, Stripe, ...), the respective `Task` (e.g., the
`ChargeWithStripeTask`) should return a `PaymentTransaction` model. This model, in turn,
should contain respective information about the transaction that was made. For example, it contains the `status` (e.g., "paid")
or a flag indicating if the `transaction` was successfully handled (`is_successful`).

Note that some information about the transaction are "pre-filled" by the `PaymentGateway` itself, so you don't need
to manually do this in your `CargeWithXTask`:

- `user_id` : The `User` who started this transaction (i.e., the buyer)
- `gateway` : The gateway that was used for this transaction (e.g., `Stripe`)
- `amount` : The amount the user was charged
- `currency` : The currency the transaction was executed

Furthermore, the gateways may add the response from the gateway to the `data` field of the `PaymentTransaction`. The
`custom` field, however, can be filled by the developer with own "custom" data.

## Developing own Payment Gateway Containers {#developing-own-payment-gateway-containers}

:::caution Instructions
This container works out of the box perfectly but if you want to change its configs or modify the codes you MUST follow these steps:

1- Copy the container from `Vendor` to `AppSection` (or any of your custom sections) of your project  
2- Fix the namespaces  
3- Remove `apiato/payment-container` dependency from project root composer.json  
:::

The `Payment` container acts as generic foundation to "plug in" different containers that interacts with specific
payment gateways (e.g., `PayPal`, `Stripe`, ...). This section introduces, how these containers shall be implemented
in order to be used via the generic `Payment` container.

In this example, we will develop a `Foo` container, that connects to the fictional `FooPay` service. As our service may
need to have its own `Accounts`, we will first need to create respective logic to add and update our `FooAccount`.
Implement respective `AddFooAccountAction` and `UpdateFooAccountAction` based on the pre-shipped `Stripe` container.

Basically, this provides the feature for the customer to create his own `FooAccount` within your own application. These
actions must, for example, provide the functionality to store user credentials (e.g., the mail-address of the user for
this specific service). Please note that you do not need to provide a dedicated `DELETE` task (as this is handled by
the `Payment` container).

The most important functionality of your `Foo` container is how to charge the `User` with this service. You, therefore,
need to create your own `ChargeWithFooTask`. This class, however, needs to implement the `PaymentChargerInterface`
distributed via the `Payment` container. This interface, in turn, requires you to implement the `charge()` method.

This method needs to connect to the `FooService`, create the payment and return a `PaymentTransaction` model.

Finally, you need to `register` the new service. This can be done in the `Payment\Configs\vendor-payment.php` file.
For the `vendor-payment.gateways` key, add the new entry for your `Foo` Payment Gateway. This may look like this:

```php
    // ...
    'foo' => [
        'container'   => 'Foo',
        'charge_task' => \App\Containers\Foo\Tasks\ChargeWithFooTask::class,
    ],
    // ...
```

Basically, this entry points to the `charger_task` that handles, how to charge a `User` with the specific `Payment Gateway`.

That's all!

## Mocking the real payment call for Testing {#mocking-the-real-payment-call-for-testing}

```php
// mock the ChargeWithStripeService external API call
$this->mockIt(ChargeWithStripeService::class)->shouldReceive('charge')->andReturn([
   'payment_method' => 'stripe',
   'description' => $payId
]);

// mock the ChargeWithPaypalService external API call
$this->mockIt(ChargeWithPaypalService::class)->shouldReceive('charge')->andReturn([
   'payment_method' => 'paypal',
   'description' => $payId
]);
```

Checkout the [Tests Helpers](../../miscellaneous/tests-helpers) page for more information about Testing.

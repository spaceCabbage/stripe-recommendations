# How I Stay Sane Implementing Stripe

_Adapted for Django/DRF with SQLite and a Vue3 Frontend_

> This README adapts [t3dogg's](https://github.com/t3dotgg) approach to Stripe subscriptions for a Django/DRF backend that uses SQLite (via Django’s ORM) instead of a KV store, along with a Vue3 frontend.
>
> The goal is to keep your Stripe state in sync with your app and avoid “split brain” race conditions.
>
> It also ensures that you always create a Stripe customer before checkout while centralizing all subscription state updates.

## Table of Contents

- [How I Stay Sane Implementing Stripe](#how-i-stay-sane-implementing-stripe)
  - [Table of Contents](#table-of-contents)
  - [Overview](#overview)
  - [Prerequisites](#prerequisites)
  - [Backend Setup](#backend-setup)
    - [Models](#models)
    - [Utility Functions](#utility-functions)
      - [Get or Create Stripe Customer](#get-or-create-stripe-customer)
      - [Sync Stripe Subscription Data](#sync-stripe-subscription-data)
    - [API Endpoints](#api-endpoints)
      - [Create Checkout Session](#create-checkout-session)
      - [Success Endpoint](#success-endpoint)
      - [Stripe Webhook](#stripe-webhook)
  - [Vue Frontend Integration](#vue-frontend-integration)
    - [Overview of the Vue3 Frontend Flow](#overview-of-the-vue3-frontend-flow)
    - [Example Vue3 component snippet:](#example-vue3-component-snippet)
    - [What Happens After the Button Is Clicked?](#what-happens-after-the-button-is-clicked)
  - [Final Considerations](#final-considerations)
    - [Authentication \& Security:](#authentication--security)
    - [Database:](#database)
    - [Testing:](#testing)
    - [Error Handling:](#error-handling)
  - [Things Still Your Problem](#things-still-your-problem)

## Overview

Stripe’s inherent “split brain” can lead to inconsistencies between Stripe’s internal state and your application’s state. The key to avoiding these issues is to:

1. **Always create (or retrieve) a Stripe customer before checkout.**
2. **Centralize subscription state updates** via a single function that syncs Stripe data into your own database.

This guide details how to implement that flow in Django/DRF using SQLite, with endpoints that your Vue3 frontend can interact with.

## Prerequisites

- **Backend Requirements:**
  - Python, Django, and Django REST Framework
  - [stripe](https://pypi.org/project/stripe/) Python library
  - SQLite (default for Django, but you can use any supported DB)
- **Frontend Requirements:**
  - Vue3
  - [Stripe.js](https://stripe.com/docs/js)
- **Environment Variables & Settings:**
  - `STRIPE_SECRET_KEY` (your Stripe secret key)
  - `STRIPE_WEBHOOK_SECRET` (for webhook signature verification)
  - `STRIPE_PRICE_ID` (price ID for the subscription)
  - `DOMAIN` (base URL for your app, e.g., `https://myapp.com`)
- A working user authentication system on your backend.

## Backend Setup

### Models

Create models to map your Django users to their Stripe customer IDs and to store subscription data. For example:

```python
# models.py
from django.conf import settings
from django.db import models

class StripeCustomer(models.Model):
    user = models.OneToOneField(settings.AUTH_USER_MODEL, on_delete=models.CASCADE)
    stripe_customer_id = models.CharField(max_length=255, unique=True)

    def __str__(self):
        return f"{self.user} - {self.stripe_customer_id}"

class StripeSubscription(models.Model):
    customer = models.OneToOneField(StripeCustomer, on_delete=models.CASCADE)
    subscription_id = models.CharField(max_length=255, null=True, blank=True)
    status = models.CharField(max_length=50)  # Consider using choices for known statuses
    price_id = models.CharField(max_length=255, null=True, blank=True)
    current_period_start = models.DateTimeField(null=True, blank=True)
    current_period_end = models.DateTimeField(null=True, blank=True)
    cancel_at_period_end = models.BooleanField(default=False)
    payment_method_brand = models.CharField(max_length=50, null=True, blank=True)
    payment_method_last4 = models.CharField(max_length=10, null=True, blank=True)

    def __str__(self):
        return f"{self.customer.user} - {self.status}"
```

### Utility Functions

Create helper functions to:

1. Get or create a Stripe customer linked to your Django user.
2. Sync subscription data from Stripe into your database.

#### Get or Create Stripe Customer

```python
# utils.py
import stripe
from django.conf import settings
from .models import StripeCustomer

stripe.api_key = settings.STRIPE_SECRET_KEY

def get_or_create_stripe_customer(user):
    try:
        stripe_customer = StripeCustomer.objects.get(user=user)
    except StripeCustomer.DoesNotExist:
      # Create a new Stripe customer with metadata linking back to your user
        customer = stripe.Customer.create(
            email=user.email,
            metadata={'user_id': user.id}
        )
        stripe_customer = StripeCustomer.objects.create(
            user=user,
            stripe_customer_id=customer['id']
        )
    return stripe_customer
```

#### Sync Stripe Subscription Data

This function fetches the latest subscription data from Stripe and updates (or creates) a record in your database.

```python
# utils.py (continued)
from datetime import datetime
from .models import StripeSubscription, StripeCustomer
from django.utils import timezone

def sync_stripe_subscription(stripe_customer_id):
    subscriptions = stripe.Subscription.list(
        customer=stripe_customer_id,
        limit=1,
        status="all",
        expand=["data.default_payment_method"],
    )

    try:
        stripe_customer = StripeCustomer.objects.get(stripe_customer_id=stripe_customer_id)
    except StripeCustomer.DoesNotExist:
        return None

    if not subscriptions.data:
        # No active subscriptions; update/create record with a "none" status
        sub_obj, _ = StripeSubscription.objects.update_or_create(
            customer=stripe_customer,
            defaults={
                "status": "none",
                "subscription_id": None,
                "price_id": None,
                "current_period_start": None,
                "current_period_end": None,
                "cancel_at_period_end": False,
                "payment_method_brand": None,
                "payment_method_last4": None,
            }
        )
        return sub_obj

    subscription = subscriptions.data[0]

    # Convert Unix timestamps to tz aware datetime objects
    current_period_start = timezone.make_aware(datetime.fromtimestamp(subscription.current_period_start))
    current_period_end = timezone.make_aware(datetime.fromtimestamp(subscription.current_period_end))


    payment_method = subscription.get("default_payment_method")
    if payment_method and isinstance(payment_method, dict):
        card = payment_method.get("card", {})
        brand = card.get("brand")
        last4 = card.get("last4")
    else:
        brand, last4 = None, None

    sub_data = {
        "subscription_id": subscription.id,
        "status": subscription.status,
        "price_id": subscription["items"]["data"][0]["price"]["id"] if subscription["items"]["data"] else None,
        "current_period_start": current_period_start,
        "current_period_end": current_period_end,
        "cancel_at_period_end": subscription.cancel_at_period_end,
        "payment_method_brand": brand,
        "payment_method_last4": last4,
    }

    sub_obj, _ = StripeSubscription.objects.update_or_create(
        customer=stripe_customer,
        defaults=sub_data
    )
    return sub_obj
```

### API Endpoints

Using Django REST Framework, create endpoints for the checkout flow, a success redirect, and handling webhooks from Stripe.

#### Create Checkout Session

This endpoint ensures the user has a Stripe customer, then creates a checkout session.

```python
# views.py
import logging
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework.permissions import IsAuthenticated
from django.conf import settings
import stripe
from .utils import get_or_create_stripe_customer

logger = logging.getLogger(__name__)
stripe.api_key = settings.STRIPE_SECRET_KEY

class CreateCheckoutSessionView(APIView):
    permission_classes = [IsAuthenticated]

    def post(self, request, format=None):
        user = request.user
        try:
            stripe_customer = get_or_create_stripe_customer(user)
        except Exception as e:
            logger.error(f"Error getting or creating Stripe customer for user {user.id}: {e}")
            return Response({"error": "Could not get or create Stripe customer."}, status=500)

        try:
            checkout_session = stripe.checkout.Session.create(
                customer=stripe_customer.stripe_customer_id,
                payment_method_types=["card"],
                line_items=[{
                    'price': settings.STRIPE_PRICE_ID,
                    'quantity': 1,
                }],
                mode='subscription',
                success_url=settings.DOMAIN + '/success',
                cancel_url=settings.DOMAIN + '/cancel',
            )
        except Exception as e:
            logger.error(f"Error creating checkout session for user {user.id}: {e}")
            return Response({"error": str(e)}, status=400)

        return Response({"sessionId": checkout_session.id})
```

Wire this view into your URL configuration so your Vue3 app can call it when the user clicks “Subscribe.”

#### Success Endpoint

This endpoint is called after a successful checkout to sync subscription data before redirecting the user.

```python
# views.py (continued)
import logging
from django.views import View
from django.http import HttpResponseRedirect
from .utils import sync_stripe_subscription, get_or_create_stripe_customer

logger = logging.getLogger(__name__)

class SuccessView(View):
    def get(self, request, *args, **kwargs):
        if not request.user.is_authenticated:
            logger.warning("Unauthenticated access to success endpoint.")
            return HttpResponseRedirect("/")
        try:
            stripe_customer = get_or_create_stripe_customer(request.user)
            sync_stripe_subscription(stripe_customer.stripe_customer_id)
        except Exception as e:
            logger.error(f"Error syncing subscription data for user {request.user.id}: {e}")
            # Optionally, redirect to a dedicated error page
            return HttpResponseRedirect("/error")
        return HttpResponseRedirect("/")  # Redirect to dashboard/homepage

```

#### Stripe Webhook

The webhook endpoint processes allowed events to update subscription data. Ensure that the raw request body is preserved for signature verification.

```python
# views.py (continued)
from rest_framework.views import APIView
from django.views.decorators.csrf import csrf_exempt
from django.utils.decorators import method_decorator
from rest_framework import status
from rest_framework.response import Response
import stripe
from django.conf import settings

ALLOWED_EVENTS = [
    "checkout.session.completed",
    "customer.subscription.created",
    "customer.subscription.updated",
    "customer.subscription.deleted",
    "customer.subscription.paused",
    "customer.subscription.resumed",
    "customer.subscription.pending_update_applied",
    "customer.subscription.pending_update_expired",
    "customer.subscription.trial_will_end",
    "invoice.paid",
    "invoice.payment_failed",
    "invoice.payment_action_required",
    "invoice.upcoming",
    "invoice.marked_uncollectible",
    "invoice.payment_succeeded",
    "payment_intent.succeeded",
    "payment_intent.payment_failed",
    "payment_intent.canceled",
]

@method_decorator(csrf_exempt, name='dispatch')
class StripeWebhookView(APIView):
    authentication_classes = []
    permission_classes = []

    def post(self, request, *args, **kwargs):
        payload = request.body
        sig_header = request.META.get('HTTP_STRIPE_SIGNATURE')
        webhook_secret = settings.STRIPE_WEBHOOK_SECRET

        try:
            event = stripe.Webhook.construct_event(
                payload, sig_header, webhook_secret
            )
        except ValueError as e:
            logger.error(f"Invalid payload: {e}")
            return Response(status=status.HTTP_400_BAD_REQUEST)
        except stripe.error.SignatureVerificationError as e:
            logger.error(f"Invalid signature: {e}")
            return Response(status=status.HTTP_400_BAD_REQUEST)
        except Exception as e:
            logger.error(f"Unexpected error during webhook verification: {e}")
            return Response(status=status.HTTP_400_BAD_REQUEST)

        try:
            if event['type'] in ALLOWED_EVENTS:
                data_object = event['data']['object']
                customer_id = data_object.get("customer")
                if customer_id and isinstance(customer_id, str):
                    sync_stripe_subscription(customer_id)
        except Exception as e:
            logger.error(f"Error processing event {event.get('id', 'unknown')}: {e}")
            return Response(status=status.HTTP_400_BAD_REQUEST)

        return Response({"received": True})

```

> Note: For Stripe webhooks, ensure your middleware does not interfere with the raw request body needed for signature verification.

## Vue Frontend Integration

### Overview of the Vue3 Frontend Flow

1. User Clicks the Subscribe Button:

   - When the user clicks the "Subscribe" button, a Vue3 component method (e.g., subscribe) is triggered.

2. API Call to Create Checkout Session:

   - The subscribe method sends a POST request to the /api/create-checkout-session/ endpoint on your Django backend.
   - The backend ensures the user has an associated Stripe customer record (creating one if necessary) and creates a Stripe Checkout session using the Stripe API.

3. Stripe Checkout Redirection:

   - The backend returns the Checkout session ID.
   - The Vue3 component then uses the Stripe.js library (via stripe.redirectToCheckout) to redirect the user to the Stripe-hosted checkout page.
   - During this redirection, the user enters payment details and completes the payment process.

4. Post-Payment Redirection to Success Endpoint:

   - After the payment is successfully processed, Stripe redirects the user to a pre-defined success URL (e.g., /success).
   - The Success endpoint in Django is invoked, which performs two key tasks:
     - It retrieves or creates the Stripe customer record for the user.
     - It calls the sync_stripe_subscription function to update the subscription status in your database.
   - If the sync is successful, the user is redirected to the home page or a dashboard; if an error occurs, they might be redirected to an error page.

5. User Sees Updated Subscription State:
   - Once redirected, the user’s subscription state is up-to-date in your application's database.
   - Your app can then display the subscription status or related information, ensuring consistency with Stripe's state.

### Example Vue3 component snippet:

```vue
<template>
  <div>
    <button @click="subscribe" :disabled="loading">
      <span v-if="loading">Processing...</span>
      <span v-else>Subscribe</span>
    </button>
    <div v-if="errorMessage" class="error">{{ errorMessage }}</div>
  </div>
</template>

<script setup>
import axios from 'axios';
import { ref } from 'vue';
import { loadStripe } from '@stripe/stripe-js';

const stripePromise = loadStripe('your-publishable-key');
const loading = ref(false);
const errorMessage = ref('');

async function subscribe() {
  loading.value = true;
  errorMessage.value = '';
  try {
    const response = await axios.post('/api/create-checkout-session/');
    const sessionId = response.data.sessionId;
    const stripe = await stripePromise;
    const { error } = await stripe.redirectToCheckout({ sessionId });
    if (error) {
      console.error(error);
      errorMessage.value =
        error.message || 'An error occurred during redirection.';
    }
  } catch (err) {
    console.error(err);
    errorMessage.value =
      'An error occurred while initiating checkout. Please try again.';
  } finally {
    loading.value = false;
  }
}
</script>

<style scoped>
.error {
  color: red;
  margin-top: 10px;
}
</style>
```

> Make sure to replace 'your-publishable-key' with your actual Stripe publishable key and adjust API endpoints as needed.

### What Happens After the Button Is Clicked?

- Button Click:

  The user initiates the process by clicking the “Subscribe” button.

- Checkout Session Creation:

  The Vue component calls the /api/create-checkout-session/ endpoint, which ensures the Stripe customer exists and creates a checkout session.

  - If successful, the session ID is returned.
  - If an error occurs, an error message is displayed on the frontend.

- Redirecting to Stripe:

  Using Stripe.js, the component redirects the user to the Stripe Checkout page.

  - Here, the user completes the payment process.
  - If redirection fails, the error is logged and shown to the user.

- Post-Payment Flow:

  - After a successful payment, Stripe redirects the user to the /success endpoint on your Django backend.
  - The backend synchronizes the subscription data and then redirects the user to the appropriate page (e.g., dashboard or home).
  - Any errors during this process are logged and may trigger a redirect to an error page.

## Final Considerations

This comprehensive error handling and clear frontend workflow ensure that both developers and users have a robust and understandable process when integrating and using Stripe subscriptions.

### Authentication & Security:

Secure your DRF endpoints with the appropriate authentication and permissions. The webhook endpoint must be publicly accessible yet secured by signature verification.

### Database:

This approach uses SQLite (via Django’s ORM). It works with any Django‑supported database if you decide to upgrade.

### Testing:

Test your webhook endpoint locally (using tools like ngrok) and in staging before deploying to production.

### Error Handling:

Implement proper logging and error handling (e.g., for network errors, missing customer IDs, or duplicate checkout attempts).

## Things Still Your Problem

While this guide covers the core flow, you’ll still need to manage:

- Environment variables for different environments (development vs production)
- Multiple subscription tiers (managing different STRIPE_PRICE_IDs)
- Exposing subscription data to users via additional endpoints
- Usage tracking, free trials, and other business-specific logic

---

> Enjoy building your SaaS with a more streamlined Stripe integration!

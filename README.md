# FreshPlate — Event Tracking Plan

**A sample analytics tracking plan for a meal-delivery subscription app.**
Designed as a portfolio piece to demonstrate event taxonomy design, naming conventions, and analytics instrumentation for Mixpanel / Segment.

> *FreshPlate is a fictional product created for this example. No real or proprietary data is used.*

---

## 1. Purpose & scope

This document defines the analytics events, properties, and identity model for **FreshPlate**, a subscription meal-delivery app where users browse weekly menus, build a box, subscribe to a plan, and rate meals.

The goal of this tracking plan is to answer core product and growth questions:
- Where do users drop off in the signup → first-order funnel?
- What drives activation (first successful delivery) and early retention?
- Which menu/merchandising elements drive add-to-box and conversion?
- What are the leading indicators of churn for a subscription product?

---

## 2. Design principles

A good tracking plan is consistent, readable, and intentional. The conventions below are applied throughout:

- **Naming:** events use `Object + Action` in Title Case with a past-tense verb — e.g. `Meal Added`, `Subscription Started`. This reads naturally in Mixpanel and keeps events grouped by object.
- **Properties:** `snake_case`, descriptive, typed consistently across events (a `plan_type` is always the same set of values everywhere it appears).
- **Track only what's actionable:** every event maps to a product or growth question. No "track everything" noise.
- **Identity:** anonymous activity is captured pre-signup and stitched to the user on signup (see Section 5).
- **Properties over events:** variations are captured as properties, not as separate events (e.g. one `Meal Added` event with a `source` property, not `Meal Added From Menu` + `Meal Added From Search`).

---

## 3. Identity model

| Concept | Implementation | Notes |
|---|---|---|
| Anonymous user | `anonymous_id` (auto, device-level) | Tracks pre-signup browsing |
| Identified user | `identify(user_id)` on signup/login | Stitches anonymous history to the user |
| Group (subscription household) | `group(account_id)` | For shared/family plans |

**Super properties** (attached to every event once known): `plan_type`, `subscription_status`, `signup_source`, `app_version`, `platform`.

---

## 4. Event catalog

Events are grouped by funnel stage. Each event lists its key properties.

### 4.1 Acquisition & onboarding

**`Signup Started`** — user begins the signup flow
- `signup_source` (organic, paid_search, referral, social)
- `entry_screen`

**`Signup Completed`** — account successfully created
- `signup_method` (email, google, apple)
- `time_to_complete_sec`

**`Onboarding Step Viewed`** — each step of onboarding
- `step_name` (dietary_prefs, delivery_zip, plan_select)
- `step_index`

**`Dietary Preferences Set`** — user sets food preferences
- `preferences` (array: vegetarian, high_protein, etc.)

### 4.2 Menu browsing & merchandising

**`Menu Viewed`** — user views a weekly menu
- `week_of`
- `meals_available_count`

**`Meal Viewed`** — user opens a meal detail
- `meal_id`, `meal_name`, `cuisine_type`
- `source` (menu, search, recommendation, collection)

**`Meal Added`** — meal added to the box
- `meal_id`, `cuisine_type`
- `source`
- `box_size_after`

**`Meal Removed`** — meal removed from the box
- `meal_id`, `box_size_after`

### 4.3 Conversion & subscription

**`Plan Selected`** — user selects a subscription plan
- `plan_type` (2_meals, 4_meals, 6_meals)
- `price_per_week`

**`Checkout Started`**
- `box_size`, `cart_value`

**`Subscription Started`** — first paid subscription created ⭐ *activation-adjacent*
- `plan_type`, `price_per_week`
- `promo_code` (nullable)
- `payment_method`

**`Order Placed`** — a weekly box is confirmed
- `order_id`, `box_size`, `order_value`
- `is_first_order` (boolean)

### 4.4 Delivery & post-purchase

**`Order Delivered`** — box delivered ⭐ *core activation event for first order*
- `order_id`, `is_first_order`
- `delivery_delay_min`

**`Meal Rated`** — user rates a delivered meal
- `meal_id`, `rating` (1–5)
- `has_review` (boolean)

### 4.5 Retention & lifecycle

**`Subscription Paused`**
- `pause_reason`, `weeks_paused`

**`Subscription Cancelled`** ⭐ *churn event*
- `cancel_reason` (price, variety, schedule, quality, other)
- `weeks_active_before_cancel`
- `ltv_at_cancel`

**`Subscription Reactivated`**
- `weeks_since_cancel`

**`Skip Week`** — user skips a delivery week
- `week_of`, `skip_reason`

---

## 5. Core funnels & metrics this enables

**Activation funnel:**
`Signup Completed` → `Plan Selected` → `Subscription Started` → `Order Placed` → `Order Delivered`

**Engagement loop:**
`Menu Viewed` → `Meal Added` → `Order Placed` → `Meal Rated`

**Key metrics unlocked:**
- Signup → first-order conversion rate (and drop-off by step)
- Time-to-activation (signup → first delivery)
- Box-building behavior: avg meals added, add/remove ratio, source attribution
- Weekly retention & skip-rate cohorts by `plan_type` and `signup_source`
- Churn analysis by `cancel_reason` and `weeks_active_before_cancel`

---

## 6. Example: properties as a table (Mixpanel-ready)

| Event | Key properties | Question it answers |
|---|---|---|
| `Subscription Started` | plan_type, promo_code, payment_method | What converts trials to paid? |
| `Order Delivered` | is_first_order, delivery_delay_min | Does delivery reliability affect retention? |
| `Meal Added` | source, cuisine_type | Which merchandising surfaces drive the box? |
| `Subscription Cancelled` | cancel_reason, weeks_active_before_cancel | Why and when do users churn? |

---

## 7. Implementation notes

- Instrument via **Segment** as the single source of truth, fanning out to Mixpanel (product analytics), a warehouse (Snowflake/BigQuery), and marketing tools.
- Validate events with a tool like the Segment debugger or a Bruno/Postman check against the tracking API before release.
- Maintain this plan as a living document (versioned), with a clear owner, so events don't drift over time.

---

*Prepared by Miguel Scagliotti Olmedo — sample portfolio piece. FreshPlate is fictional; no proprietary data is included.*

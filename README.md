# FreshPlate — Event Tracking & Metadata Plan

**A sample analytics tracking plan and event-metadata reference for a meal-delivery subscription product (web + app).**
Demonstrates event taxonomy design, metadata documentation, naming conventions, and analytics instrumentation for Mixpanel / Segment — structured for both human analysts and AI/LLM consumption.

> *FreshPlate is a fictional product created for this example. No real or proprietary data is used.*

---

## 1. Purpose & scope

This document defines the analytics events, event metadata, properties, and identity model for **FreshPlate**, a subscription meal-delivery product where users browse weekly menus on the **website and app**, build a box, subscribe to a plan, and rate meals.

The goal is to produce documentation that is:
- **Clear** enough for a non-technical stakeholder to understand what each event means.
- **Precise** enough for engineering to implement consistently.
- **Structured** enough to feed analytics tools and AI/LLM applications (see Section 8).

It answers core product and growth questions:
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
- **Self-describing metadata:** every event and property carries a plain-language description, a type, and allowed values — so the documentation is usable without tribal knowledge.

---

## 3. Identity model

| Concept | Implementation | Notes |
|---|---|---|
| Anonymous user | `anonymous_id` (auto, device-level) | Tracks pre-signup browsing |
| Identified user | `identify(user_id)` on signup/login | Stitches anonymous history to the user |
| Group (subscription household) | `group(account_id)` | For shared/family plans |

**Super properties** (attached to every event once known): `plan_type`, `subscription_status`, `signup_source`, `app_version`, `platform` (web / ios / android).

---

## 4. Web events (site-level tracking)

These capture website behavior before and during the funnel — the foundation of web event tracking.

**`Page Viewed`** — any page load
- `page_name`, `page_url`, `referrer`
- `utm_source`, `utm_medium`, `utm_campaign` (nullable)

**`CTA Clicked`** — click on a primary call-to-action
- `cta_label`, `cta_location` (hero, nav, footer, sticky_bar)
- `destination_url`

**`Scroll Depth Reached`** — engagement on long pages (e.g. landing/menu)
- `page_name`, `depth_pct` (25, 50, 75, 100)

**`Form Submitted`** — any form submission (lead capture, zip check, contact)
- `form_name` (zip_check, lead_capture, contact)
- `success` (boolean), `error_message` (nullable)

**`Search Performed`** — on-site search
- `query`, `results_count`

---

## 5. Product events by funnel stage

### 5.1 Acquisition & onboarding
**`Signup Started`** — `signup_source`, `entry_screen`
**`Signup Completed`** — `signup_method` (email, google, apple), `time_to_complete_sec`
**`Onboarding Step Viewed`** — `step_name` (dietary_prefs, delivery_zip, plan_select), `step_index`
**`Dietary Preferences Set`** — `preferences` (array)

### 5.2 Menu browsing & merchandising
**`Menu Viewed`** — `week_of`, `meals_available_count`
**`Meal Viewed`** — `meal_id`, `meal_name`, `cuisine_type`, `source` (menu, search, recommendation, collection)
**`Meal Added`** — `meal_id`, `cuisine_type`, `source`, `box_size_after`
**`Meal Removed`** — `meal_id`, `box_size_after`

### 5.3 Conversion & subscription
**`Plan Selected`** — `plan_type` (2_meals, 4_meals, 6_meals), `price_per_week`
**`Checkout Started`** — `box_size`, `cart_value`
**`Subscription Started`** ⭐ — `plan_type`, `price_per_week`, `promo_code` (nullable), `payment_method`
**`Order Placed`** — `order_id`, `box_size`, `order_value`, `is_first_order` (boolean)

### 5.4 Delivery & post-purchase
**`Order Delivered`** ⭐ — `order_id`, `is_first_order`, `delivery_delay_min`
**`Meal Rated`** — `meal_id`, `rating` (1–5), `has_review` (boolean)

### 5.5 Retention & lifecycle
**`Subscription Paused`** — `pause_reason`, `weeks_paused`
**`Subscription Cancelled`** ⭐ — `cancel_reason`, `weeks_active_before_cancel`, `ltv_at_cancel`
**`Subscription Reactivated`** — `weeks_since_cancel`
**`Skip Week`** — `week_of`, `skip_reason`

---

## 6. Event metadata reference (detailed example)

Full metadata specification for a single event — the level of detail applied to every event in a production plan. This is the format an engineer implements against and an analyst (or AI tool) reads.

### Event: `Subscription Started`
**Description:** Fired when a user successfully creates their first paid subscription. Core conversion/activation event.
**Trigger:** Server-side, on successful payment authorization.
**Owner:** Growth squad

| Property | Type | Required | Allowed values / format | Description |
|---|---|---|---|---|
| `plan_type` | string | yes | `2_meals` \| `4_meals` \| `6_meals` | Subscription tier selected |
| `price_per_week` | number | yes | decimal, USD | Weekly price at time of subscription |
| `promo_code` | string | no | uppercase alphanumeric, nullable | Promo applied, if any |
| `payment_method` | string | yes | `card` \| `paypal` \| `apple_pay` | Method used |
| `signup_source` | string | yes | `organic` \| `paid_search` \| `referral` \| `social` | Original acquisition channel |
| `is_trial_conversion` | boolean | yes | `true` \| `false` | Whether this converted from a trial |

---

## 7. Core funnels & metrics this enables

**Web-to-signup funnel:** `Page Viewed` → `CTA Clicked` → `Signup Started` → `Signup Completed`

**Activation funnel:** `Signup Completed` → `Plan Selected` → `Subscription Started` → `Order Placed` → `Order Delivered`

**Engagement loop:** `Menu Viewed` → `Meal Added` → `Order Placed` → `Meal Rated`

**Key metrics unlocked:** web→signup conversion and drop-off by step; signup→first-order rate; time-to-activation; box-building behavior; weekly retention & skip-rate cohorts by `plan_type` and `signup_source`; churn analysis by `cancel_reason`.

---

## 8. AI / LLM-ready documentation notes

This plan is structured so it can be consumed not only by analysts but by AI applications (e.g. an internal assistant that answers "what does event X mean?" or auto-generates analytics queries):

- **Consistent, predictable schema** — every event has the same fields (description, trigger, owner, typed properties with allowed values), so a model can parse it reliably.
- **Self-describing names & descriptions** — no abbreviations or tribal knowledge; each event/property is understandable in isolation.
- **Controlled vocabularies** — allowed-values lists (e.g. `cancel_reason`) give a model a closed set to reason over, reducing hallucination.
- **Stable identifiers** — `order_id`, `user_id`, `meal_id` are explicit, so relationships between events are machine-traceable.

*(In a prior role I applied this same principle — structuring documentation in Markdown as a single source of truth — to build a knowledge base that trained an internal AI support assistant, reducing repetitive queries.)*

---

## 9. Implementation notes

- Instrument via **Segment** as the single source of truth, fanning out to Mixpanel (product analytics), a warehouse (Snowflake/BigQuery), and marketing tools.
- Validate events with the Segment debugger or a Bruno/Postman check against the tracking API before release.
- Maintain this plan as a **living, versioned document** with a clear owner, so events don't drift over time.

---

*Prepared by Miguel Scagliotti Olmedo — sample portfolio piece. FreshPlate is fictional; no proprietary data is included.*

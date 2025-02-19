---
title: "E-commerce"
sidebar_position: 105
hide_title: true
---

```mdx-code-block
import Badges from '@site/src/components/Badges';
```
<Badges badgeType="dbt-package Release" pkg="ecommerce"></Badges>

# Snowplow E-commerce Package

**The package source code can be found in the [snowplow/dbt-snowplow-ecommerce repo](https://github.com/snowplow/dbt-snowplow-ecommerce), and the docs for the [model design here](https://snowplow.github.io/dbt-snowplow-ecommerce/#!/overview/snowplow_ecommerce).**

The package contains a fully incremental model that transforms raw web e-commerce event data generated by the [Snowplow JavaScript tracker](/docs/collecting-data/collecting-from-own-applications/javascript-trackers/index.md) into a set of derived tables based around the following e-commerce data objects: carts, checkouts, products and transactions.

The Snowplow e-commerce data model aggregates Snowplow's out of the box `snowplow_ecommerce_action` events to create a set of derived tables based on the e-commerce data objects - carts, checkouts, products, transactions, users - that contain many useful dimensions as well as calculated measures.

## Overview

The package contains multiple staging models however the mart models are as follows:

- Base: Performs the incremental logic, outputting the table `snowplow_ecommerce_base_events_this_run` which contains a de-duped data set of all events required for the current run of the model, and is the foundation for all other models generated.
- Carts: Parses the cart interactions that occur to provide handy filters and aggregations, which helps identify what happened to carts on a session level to extract, for example, abandoned carts with ease. The Carts module outputs the tables `snowplow_ecommerce_cart_interactions`.
- Checkouts: Parses checkout steps that occur to provide handy filters and aggregations to help identify which checkout steps were walked through, and what details were entered in each of these steps. This lends itself well to a funnel analysis. The Checkouts module outputs the `snowplow_ecommerce_checkout_interactions` table.
- Products: Parses product view and list information to provide insights into which products were being viewed, what details were being shown to the end user and how the user then interacted with these products. The Products module outputs the `snowplow_ecommerce_product_interactions` table.
- Transactions: Parses transaction actions to provide insights into which transactions occurred, how much revenue was generated from them, and other insights leveraging the many properties of the transaction e-commerce context. The Transactions module outputs the `snowplow_ecommerce_transaction_interactions` table.
- Sessions: Aggregates all other data into a sessions table which leverages the  `domain_sessionid`, outputting the `snowplow_ecommerce_sessions` table.

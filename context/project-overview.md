# Project Overview

## Overview

This application is a no-login web dashboard that scans financial markets over a rolling one-week period, explains what happened and why in a short AI-generated summary, and shows where capital is currently allocated across major asset classes (oil, gold, stocks by domain, US/EU government bonds, and crypto). It exists for investors who want a fast, plain-language read on current market conditions and the news driving them, without piecing that story together themselves from scattered sources.

## Goals

1. Give investors a fast, plain-language read of the current market state for a defined weekly period.
2. Show which news events are driving that market state, with direct links to sources.
3. Show current capital allocation across major asset categories (oil, gold, stocks, bonds, crypto) with approximate market cap and percentage share.
4. Make every AI-generated summary auditable — a human can trace it back to the exact news and price data used to produce it.
5. Ship a lean, no-login first version that any visitor can use with zero setup.

## Core User Flow

1. User opens the website. No sign-up, login, or account creation is required.
2. The current period (e.g., "Sep 1 – Sep 7, 2026") is shown at the top of the page.
3. In the center of the page, the user reads the market state summary — a short, AI-generated explanation of what happened in the market during that period and why.
4. On the left side, the user sees a simple, unfiltered list of the major news items that led to that market state.
5. Alongside the summary, the user sees capital allocation broken out by category — oil, gold, stocks by domain, EU/US government bonds, crypto — each with approximate market capitalization and percentage share.
6. The user has reached the core value of the product without any further clicks, filters, or setup.

## Features by Category

### Market State Summary
- Short, AI-generated explanation of the current market state for the active one-week period.
- Displayed centrally as the primary focal point of the page.

### News Feed
- Simple list of major news items relevant to tracked assets and top global events for the period.
- No search, filtering, or sorting controls — a flat list only.
- Displayed on the left side of the layout.

### Capital Allocation
- Breakdown by category: oil, gold, stocks (by domain), government bonds (EU and US), crypto.
- Each category shows approximate market capitalization and percentage share.

### Data Pipeline (Backend)
- Fetches news for the period, covering tracked assets plus top global stories.
- Fetches price data per asset at daily/hourly granularity for the period, giving the AI enough resolution to correlate price moves with news events.
- News fetching and price fetching have no dependency on each other and can be built in parallel.

### AI Summary Engine
- Takes the stored news and price data as context.
- Produces a short, meaningful natural-language summary of market changes and current state for the period.
- Depends on the data pipeline completing first.

### Verification & Audit Trail
- Every generated summary is saved together with the exact list of news items and price data that were fed to the AI.
- Links the saved summary to its source news and to price data as of the last day of the period.
- Lets a human check why the AI said what it said.

## In Scope

- Displaying the current market state as a short AI-generated summary for a one-week period.
- Displaying the news that led to that market state, as a simple, unfiltered, unsorted list.
- Displaying capital allocation across oil, gold, stocks (by domain), EU/US government bonds, and crypto, with market cap and percentage share.
- Fetching and storing news and price data via APIs.
- Running an AI model (e.g., a ChatGPT-class model) over the stored news and price data to generate the summary.
- Saving each generated summary with references to the underlying news and price data used, so it can be verified by a human.
- A basic no-login UI built with Next.js, React, Tailwind CSS, and shadcn/ui components, using a simple dark theme.
- PostgreSQL as the primary data store, hosted on a managed platform such as Neon or Supabase.
- A caching layer to reduce repeated calls to news/price APIs and the AI model.

## Out of Scope

- Predictions or forecasts of future market moves — the product only explains what already happened.
- Sign-up and sign-in flows of any kind.
- Account management (profiles, settings, saved preferences, etc.).
- Complex news search, filtering, or sorting — the news list is intentionally a simple, plain list with no controls.
- Browsing or comparing historical periods beyond the current one-week window. *(Assumption — not explicitly stated as out of scope, but not part of the described flow either. Flag this if multi-period browsing is actually needed for v1.)*

## Success Criteria

The first version is done when:

1. **News data** — The database contains news for the current period, covering tracked assets and top global stories, fetched via API.
2. **Price data** — The database contains price data per asset, at daily/hourly granularity, fetched via API, for the current period.
3. **AI summary** — An AI model is connected, fed the stored news and price data as context, and reliably produces a short, coherent summary of market state and changes.
4. **Saved & verifiable** — Each generated summary is saved as the record for its period, linked to the specific news items and to price data as of the period's last day, we should have data connection in db.
5. **UI** — The UI (period header, central market-state summary, left-side news list, categorized capital allocation view) is built and displays real data from the database.
6. **No-login access confirmed** — Any visitor can open the site and see the full experience (summary, news, allocation) with zero sign-up or authentication steps.

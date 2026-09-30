# Project Overview

## Overview

This application is a no-login web dashboard that scans financial markets over a rolling one-week period and explains the market's broad direction in a short AI-generated narrative. It compares price changes in representative market proxies—such as indexes, ETFs, futures, or other liquid benchmarks—for major asset groups: oil, gold, equities by domain, US/EU government bonds, and crypto.

The comparison is a practical signal of relative market movement and possible rotation between asset groups. The dashboard combines the proxy-price evidence with relevant news, then gives that evidence to an LLM to produce an auditable, plain-language explanation of what moved, how the asset groups moved relative to one another, and the events most likely associated with those moves.

It exists for investors who want a fast, evidence-backed read on the current market trend and the news shaping it, without assembling that story manually from separate sources.

## Goals

1. Give investors a fast, plain-language read of the market's broad direction for a defined weekly period.
2. Show relative value changes across major asset groups using transparent, price-based proxy data.
3. Use relevant news together with proxy-price movements to explain the market narrative, with direct links to source articles.
4. Make every AI-generated narrative auditable: a human can trace it to the exact proxy prices, calculated changes, and news items used.
5. Ship a lean, no-login first version that any visitor can use with zero setup.

## Core User Flow

1. User opens the website. No sign-up, login, or account creation is required.
2. The current period (for example, “Sep 1 – Sep 7, 2026”) appears at the top of the page.
3. In the center, the user reads a short AI-generated market narrative explaining the week's general trend, the strongest and weakest asset groups, and the relevant news context.
4. On the left, the user sees a simple, unfiltered list of the major news items used as evidence for the narrative.
5. Alongside the narrative, the user sees each asset group's representative proxy or proxies and its price change over the period, making relative movement between groups easy to compare.
6. The user can inspect the underlying data and source links to understand what supports the narrative.

## Features by Category

### Market Trend Narrative

- Short, AI-generated explanation of the general market trend for the active one-week period.
- Describes proxy-based relative movements across tracked asset groups and connects them to relevant news where the evidence supports that connection.
- Displayed centrally as the primary focal point of the page.

### News Feed

- Simple list of major news items relevant to tracked proxies and top global events for the period.
- No search, filtering, or sorting controls—a flat list only.
- Displayed on the left side of the layout.

### Asset-Group Movement

- Compares price changes for representative proxies across oil, gold, equities, US/EU government bonds, and crypto.
- A proxy may be an index, ETF, or another liquid market benchmark appropriate to its asset group.
- Shows each proxy's movement for the period and an asset-group-level comparison that reveals the market's relative direction.

### Data Pipeline (Backend)

- Fetches news for the period, covering tracked proxies plus top global stories.
- Fetches prices for the period, one at start the second at end.
- Normalizes the data needed to calculate period changes and compare groups consistently.
- Stores the raw price observations, calculated movements, and news records used by the narrative.
- News and price collection are independent and can be built in parallel.

### AI Narrative Engine

- Takes the stored news, proxy-price observations, and calculated period changes as context.
- Produces a short, meaningful natural-language narrative of market direction and relative asset-group movement.
- Grounds claims in the supplied evidence, avoids treating correlation as proof of causation, and does not forecast future moves.
- Depends on the data pipeline completing first.

### Verification & Audit Trail

- Every generated narrative is saved with the exact news items, proxy definitions, price observations, and calculated changes given to the AI.
- Links the narrative to source articles and to the data observed at the period boundaries.
- Lets a human verify both the observed movements and the evidence behind the narrative.

## In Scope

- Displaying the current market trend as a short AI-generated narrative for a one-week period.
- Displaying a simple, unfiltered list of news items that inform that narrative.
- Defining and showing representative price proxies for oil, gold, equities by domain, US/EU government bonds, and crypto.
- Calculating and displaying price changes for those proxies and relative movement between asset groups.
- Fetching and storing news and proxy-price data via APIs.
- Running an AI model (for example, a ChatGPT-class model) over the stored news and proxy data to generate the narrative.
- Saving each generated narrative with references to its underlying news, proxy definitions, price observations, and calculations so it can be audited by a human.
- A basic no-login UI built with Next.js, React, Tailwind CSS, and shadcn/ui components, using a simple dark theme.
- PostgreSQL as the primary data store, hosted on a managed platform such as Neon or Supabase.
- A caching layer to reduce repeated calls to news/price APIs and the AI model.

## Out of Scope

- Estimating market capitalization, static capital allocation, or percentage share by asset group.
- Predictions or forecasts of future market moves—the product only explains what already happened.
- Sign-up and sign-in flows of any kind.
- Account management (profiles, settings, saved preferences, and similar features).
- Complex news search, filtering, or sorting—the news list is intentionally a simple, plain list with no controls.
- Browsing or comparing historical periods beyond the current one-week window. *(Assumption—not explicitly stated as out of scope, but not part of the described v1 flow.)*

## Success Criteria

The first version is done when:

1. **Proxy catalog** — Each tracked asset group has one or more documented representative proxies suitable for consistent weekly comparison.
2. **News data** — The database contains news for the current period, covering tracked proxy themes and top global stories, fetched via API.
3. **Price data** — The database contains daily price data for every selected proxy for the current period, fetched via API.
4. **AI narrative** — An AI model is connected, receives the stored news and proxy-price data as context, and reliably produces a short, coherent narrative of market direction and relative asset-group movement without presenting inference as fact.
5. **Saved & verifiable** — Each generated narrative is saved as the record for its period and linked to the exact news items, proxy definitions, price observations, and calculations used to generate it.
6. **UI** — The UI (period header, central market-trend narrative, left-side news list, and asset-group movement view) displays real data from the database.
7. **No-login access confirmed** — Any visitor can open the site and see the full experience—narrative, news, and proxy-based market movement—with zero sign-up or authentication steps.

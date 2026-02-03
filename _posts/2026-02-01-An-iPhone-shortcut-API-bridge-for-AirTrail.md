---
title: An iPhone shortcut API bridge for AirTrail
layout: post
tags:
  - guides
  - API
  - iOS
  - AirTrail
category: Code
published: false
---
[AirTrail](https://github.com/johanohly/AirTrail) is an open-source, personal flight tracking system. I like seeing everywhere I've been, and places I want to go. It's self-hosted in my HomeLab and I try to keep it upto date, but sometimes forget.

The UI is quite intuitive, give it a flight number and it'll find all the details from APIs like [AeroDataBox (ADB)](https://airtrail.johan.ohly.dk/docs/integrations/aerodatabox). There's an option to add seat number, and notes which I use for the Booking Reference and Ticket Number. Inevitably, next time Lufthansa cancels my flight, I'll have the info to hand.


![flight](/assets/img/at-flight.png)


If, predictably, I forget to add my flight immediately after travelling, tracking down an old ticket can be annoying (except RyanAir, thanks). ADB API has a historical data limit of 3 days (Free tier), digging it out from other sources can take time. I hate leaving boxes empty so I'll try to find even the aircraft model.

I want a lazy proof way to add flights, without having to load up the UI, login, click around blah blah. Enter the [AirTrail API](https://airtrail.johan.ohly.dk/docs/api/flight/save-flight).

The API, however, is stricter, it doesn't let me simply enter a flight number and depature date, then scurry away to find the rest. It requires: `from` and `to` airport fields and `departureDate`. Only the [ICAO four letter code](https://en.wikipedia.org/wiki/ICAO_airport_code) is acceptable. Being a frequent flyer, I can recall [IATA three letter codes](https://en.wikipedia.org/wiki/IATA_airport_code), but four and the similarily of ICAO is a bit of niche.

For now, I'll test the API as-is, it takes a `POST` request with a `Bearer token`. First, I need to create an API key in AirTrail, **Settings → Security → create API key**.

![API](/assets/img/at-api.png)

I'll use PostMan to send a test call, using the schema from the API docs.

______

##  Baseline: calling manually
A minimal save call:

<details markdown="1">
```bash
curl --location 'http://airtrail-url/api/flight/save' \
  --header 'Content-Type: application/json' \
  --header 'Authorization: Bearer xxxx' \
  --data '{
    "from": "KJFK",
    "to": "EGLL",
    "departure": "2022-01-01",
    "seats": [
      { "userId": "<USER_ID>" }
    ]
  }'
</deatils> ```

That works - with only required fields. User ID isn't neccesary, the API fetches the User ID associated with a personal `Bearer token` from earlier.

______

## Problem: only a flight number

Flight numbers are recycled, typically for the same route, every day, departure date is essential.

**What are my options?**

I can build a small intermediary API before calling AirTrail's, but since AirTrail is an active open-source project, lets modify the API and eventually contribute back to the project. I found `getFlightRoute` in `lib/server/utils/flight-lookup/flight-lookup`, that's what we need to call.

**What about the interface?**

Some kind of chat bot would be nice, but perhaps I'm over-engineering the problem. Then I remembered, [iPhone Shortcuts](https://medium.com/@richardmoult75/how-to-use-shortcuts-to-enter-data-into-api-6983db92852b).

I can request input as variables then send to an API. My test case is to first design the shortcut, since I know my desired end-state: 

![iOSShortcut](/assets/img/at-shortcut1.jpeg)

![iOSShortcut2](/assets/img/at-shortcut2.jpeg)


The original date includes a timestamp, not compatible with the API schema, so `formatted_date` returns an schema expected `ISO6801` e.g. `2026-01-01`.

______

## Understanding: The UI Process

`getFlightRoute`:

```interface FlightLookupProvider {
  getFlightRoute: (
    flightNumber: string,
    opts?: FlightLookupOptions,
  ) => Promise<FlightLookupResult>;
}

const aerodataboxProvider: FlightLookupProvider = {
  getFlightRoute: getAerodataboxFlightRoute,
};

const adsbdbProvider: FlightLookupProvider = {
  getFlightRoute: (flightNumber: string) => getAdsbdbFlightRoute(flightNumber),
};```

Since the result type is identical for AeroDataBox and ADSB DB, that's helpful, and decision logic of which one to use is handled already.

When the method is invoked, if no date is provided, it searches in a 2-day window and returns potential matches. I must make departure date mandatory so we only return one result.

The new logic:

1. If `from` and `to` airports are defined, continue as normal regardless of other fields, call this manual entry.
2. If not defined, flight number must be defined. Find the flight for today, unless departure date is also defined, then search accordingly. 

______

## New Code: adding logic and `getFlightRoute`

In the add/create flight API:

```
  const parsed = saveApiFlightSchema.safeParse(flight);
  if (!parsed.success) {
    return json(
      { success: false, errors: parsed.error.errors },
      { status: 400 },
    );
  }
 ```

First, import getFlightRoute method and format for date transformations:

```import { getFlightRoute } from '$lib/server/utils/flight-lookup/flight-lookup';```
```import { format } from 'date-fns';```

const parsed = saveApiFlightSchema.safeParse(flight);
Add a “lookup if needed” block
This is the core logic I added (formatted cleanly here):


Design choices (and why they matter)
1) Don’t “invent” the date unless you must
If the caller provides departure, use that day for lookup. Otherwise default to today.

2) Don’t auto-fill future flights if the caller didn’t provide from/to
A flight number alone can match multiple instances. Also, future flights may not have reliable arrival values yet.

So the guardrail is:

If caller did not supply from&#x2F;to → only accept results where arrival &lt;= now (i.e., flight elapsed)
This prevents saving rubbish.

3) Only populate missing fields
If the caller supplies from&#x2F;to, don’t override them. Same for airline/aircraft fields.

4) Rate limiting
If you validate twice and you’re hammering RapidAPI, you’ll hit short-window rate limits quickly.

If you run into this:

avoid duplicate validations / lookups
cache recent lookups keyed by flightNumber+date for a few minutes
optionally return partial success (“saved with minimal fields”) if the lookup fails but required schema fields exist
Reality check: “Works in UI but not API”
If you find it works in the UI but not through the API handler, there’s a high chance:

you imported a UI-only module into the API layer, or
the lookup route expects auth/session context, or
server runtime environment differs (e.g., missing RapidAPI key in API execution context), or
you’re simply getting rate limited and the lookup silently fails
The quickest way to diagnose: log the lookup error response and headers from RapidAPI (status codes matter).

The no-touch alternative (recommended if you don’t want to fork AirTrail)
If you don’t want to modify AirTrail at all (fair), run a tiny intermediary:

1) Accept { flightNumber, date?, userId } 2) Call AeroDataBox via RapidAPI to get departure.airport.icao and arrival.airport.icao 3) Call AirTrail save-flight with those ICAO codes

That’s cleaner long-term, keeps your AirTrail instance upgradeable, and avoids mixing UI lookup logic into the API save route.

Next step: iPhone Shortcut input
The final piece is a shortcut that accepts either:

text input (BA117 2026-02-14), or
a share-sheet action (select text from an email / screenshot, then parse flight number)
Then it calls the intermediary endpoint (or the patched AirTrail API, if you went that route) and returns:


Link references (keep these, replace with your real URLs)
AirTrail Save Flight docs:
https:&#x2F;&#x2F;airtrail.johan.ohly.dk&#x2F;docs&#x2F;api&#x2F;flight&#x2F;save-flight

AeroDataBox RapidAPI OpenAPI:
https:&#x2F;&#x2F;doc.aerodatabox.com&#x2F;docs&#x2F;openapi-rapidapi-v1.json
Prose

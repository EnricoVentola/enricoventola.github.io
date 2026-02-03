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
Describe your Changes
Update 2026-02-01-An-iPhone-shortcut-API-bridge-for-AirTrail.md
Commit
EnricoVentola / enricoventola.github.io
An iPhone shortcut API bridge for AirTrail
AirTrail is an open-source, personal flight tracking system. I like seeing everywhere I've been, and places I want to go. It's self-hosted in my HomeLab and I try to keep it upto date, but sometimes I forget.

The UI is great, give it a flight number, it looks up a recent route/time via AeroDataBox (ADB) API, and you're done. There's an option to add seat number, and notes which I use for the Booking Reference and Ticket Number. Inevitably, next time Lufthansa cancels my flight, I'll have the info to hand.

at-flight.png

If I forget to add my flight immedaitely after travelling, then tracking down an old ticket can be annoying (Except RyanAir, thanks), ADB API also has a limit on info for past flights, so then digging it out from other sources can take time. I don't like leaving boxes empty, so I'll try to find even the aircraft model.

I want a lazy proof way to add a flight, without having to load up the UI, login, click around blah blah. Enter the AirTrail API.

The API, however, is stricter, it doesn't let you simply enter a flight number and depature date then scurry away to find the rest from ADB, it requires: from and to airport fields, and only the ICAO four letter code. Having flown a lot, I can recall most IATA three letter codes , but four and the similarily of ICAO is a bit of a push.

For now, I'll test the API as-is, it takes a POST request with a Bearer token. First, I need to create an API key at Settings → Security → create API key

at-api.png

I'll use PostMan to send a test call, using the schema from the API docs.

Baseline: calling manually
A minimal save call looks like this:

curl --location &#x27;http:&#x2F;&#x2F;air trail URL&#x2F;api&#x2F;flight&#x2F;save&#x27; \
  --header &#x27;Content-Type: application&#x2F;json&#x27; \
  --header &#x27;Authorization: Bearer xxxx&#x27; \
  --data &#x27;{
    &quot;from&quot;: &quot;KJFK&quot;,
    &quot;to&quot;: &quot;EGLL&quot;,
    &quot;departure&quot;: &quot;2022-01-01&quot;,
    &quot;seats&quot;: [
      { &quot;userId&quot;: &quot;&lt;USER_ID&gt;&quot; }
    ]
  }&#x27;
That works — but the requirement for ICAO codes is the pain point. User ID isn't neccesary because the API code also uses the personal Bearer token to find an associated user id.

If I’m logging flights quickly, I want to type something like:

BA117 2026-02-14
or even just BA117
Does the API save route through the same flight-lookup logic?

Short answer: no. Some kind of chat bot would be best, but perhaps over-engineered.

So either: 1) I build a small intermediary that does the lookup first, then calls AirTrail’s save API, without changing AirTrail, or
2) I modify AirTrail so the API path uses the same getFlightRoute logic as the UI.

Then I remembered, iPhone Shortcuts.

I can request user input as variables, then send an API call with those vars. I'll code my end-state in here, since I know the API already works.

The original date includes a timestamp, which isn't compatible with this API, it was bugging out the code, so formatted_date returns the format the API is expecting in ISO6801 2026-01-01.

What AirTrail’s UI actually does when you add a flight (AeroDataBox via RapidAPI)
Roughly, clicking search in the UI:

Sanitises the flight number (removes spaces)
Does Svelte routing/flow stuff
If AeroDataBox isn’t configured, it falls back to ADSBDB (leave it out for now, I have an ADB Free API key entered in the Settings menu)
If AeroDataBox is configured:
It constructs a URL
Calls RapidAPI using the internal flight-lookup route
Server returns candidate flights
UI lets you pick/whittle down the correct one
This is wrapped in a reusable server route called getFlightRoute. Good start.

The AeroDataBox URL AirTrail builds
This is what AirTrail uses to query AeroDataBox (via RapidAPI), depending on whether you supply a date:

let url: string;
if (date) {
  url = `${BASE_URL}&#x2F;flights&#x2F;number&#x2F;${encodeURIComponent(
    cleaned,
  )}&#x2F;${format(date, &#x27;yyyy-MM-dd&#x27;)}?dateLocalRole=Both&amp;withAircraftImage=false&amp;withLocation=false`;
} else {
  const now = new Date();
  const fromDate = format(subDays(now, 2), &#x27;yyyy-MM-dd&#x27;);
  const toDate = format(addDays(now, 2), &#x27;yyyy-MM-dd&#x27;);
  url = `${BASE_URL}&#x2F;flights&#x2F;number&#x2F;${encodeURIComponent(
    cleaned,
  )}&#x2F;${fromDate}&#x2F;${toDate}?dateLocalRole=Both&amp;withAircraftImage=false&amp;withLocation=false`;
}
So we already know:

searchBy = number
if no date is supplied, it searches a ±2 day window
it’s requesting minimal extra data (withAircraftImage=false, withLocation=false)
dateLocalRole=Both is used (useful, but can produce extra candidate matches)
A nuance here, I don't want an array of results in a +- 2 day window, since either I willl have just left the aircraft of the flight I want to add, or it'll be on a defined day, probably recently.

Fix approach: route API saves through the same lookup used by the UI
The UI calls a routable function getFlightRoute. So the idea is:

If save-flight validation fails and a flight number exists:
run getFlightRoute(flightNumber, { date })
pick the best result
populate missing fields (from&#x2F;to, times, airline, aircraft, etc.)
revalidate
save
Import the lookup function
import { getFlightRoute } from &#x27;$lib&#x2F;server&#x2F;utils&#x2F;flight-lookup&#x2F;flight-lookup&#x27;;
Where the API used to stop
Previously, the API basically did:

const parsed = saveApiFlightSchema.safeParse(flight);
Add a “lookup if needed” block
This is the core logic I added (formatted cleanly here):

let parsed = saveApiFlightSchema.safeParse(flight);
  let lookupAttempted = false;
  let lookupFound = false;
  let lookupNoElapsedMatch = false;

  ## If initial validation fails but flight number is provided, try lookup
  console.log(&#x27;Initial parse success:&#x27;, parsed.success);
  console.log(&#x27;body.flightNumber:&#x27;, body.flightNumber);
  console.log(&#x27;body.from:&#x27;, body.from, &#x27;body.to:&#x27;, body.to);

  if (!parsed.success &amp;&amp; body.flightNumber &amp;&amp; (!body.from || !body.to)) {
    console.log(&#x27;Attempting lookup...&#x27;);
    lookupAttempted = true;
    try {
      const opts = flight.departure
        ? { date: new Date(flight.departure.split(&#x27;T&#x27;)[0]) }
        : { date: new Date() };
      const routes = await getFlightRoute(body.flightNumber, opts);
      if (Array.isArray(routes) &amp;&amp; routes.length &gt; 0) {
        const userProvidedDeparture = !!body.departure;
        let chosen: typeof routes[0] | null = null;

        if (userProvidedDeparture) {
          chosen = routes[0] ?? null;
        } else {
          const now = Date.now();
          chosen = routes.find((r) =&gt; r.arrival &amp;&amp; r.arrival.getTime() &lt;= now) ?? null;
          if (!chosen &amp;&amp; routes.length &gt; 0) {
            lookupNoElapsedMatch = true;
          }
        }

        if (chosen) {
          flight.from = flight.from ?? chosen.from?.icao;
          flight.to = flight.to ?? chosen.to?.icao;
          flight.airline = flight.airline ?? chosen.airline?.icao ?? flight.airline;
          flight.aircraft = flight.aircraft ?? chosen.aircraft?.icao ?? flight.aircraft;
          flight.aircraftReg = flight.aircraftReg ?? chosen.aircraftReg ?? null;

          if (chosen.departure) {
            flight.departure = chosen.departure.toISOString();
            flight.departureTime = format(chosen.departure, &#x27;HH:mm&#x27;);
          }
          if (chosen.arrival) {
            flight.arrival = chosen.arrival.toISOString();
            flight.arrivalTime = format(chosen.arrival, &#x27;HH:mm&#x27;);
          }
          if (chosen.departureScheduled) {
            flight.departureScheduled = chosen.departureScheduled.toISOString();
            flight.departureScheduledTime = format(chosen.departureScheduled, &#x27;HH:mm&#x27;);
          }
          if (chosen.arrivalScheduled) {
            flight.arrivalScheduled = chosen.arrivalScheduled.toISOString();
            flight.arrivalScheduledTime = format(chosen.arrivalScheduled, &#x27;HH:mm&#x27;);
          }

          flight.departureTerminal = flight.departureTerminal ?? chosen.departureTerminal ?? null;
          flight.departureGate = flight.departureGate ?? chosen.departureGate ?? null;
          flight.arrivalTerminal = flight.arrivalTerminal ?? chosen.arrivalTerminal ?? null;
          flight.arrivalGate = flight.arrivalGate ?? chosen.arrivalGate ?? null;

          parsed = saveApiFlightSchema.safeParse(flight);
          if (parsed.success) {
            lookupFound = true;
            console.log(&#x27;Lookup succeeded, flight populated&#x27;);
          } else {
            console.log(&#x27;Lookup populated flight but re-parse failed:&#x27;, parsed.error.errors);
          }
        } else {
          console.log(&#x27;Lookup did not find suitable route. userProvidedDeparture:&#x27;, userProvidedDeparture, &#x27;routes:&#x27;, routes.length);
        }
      } else {
        console.log(&#x27;Lookup returned no routes&#x27;);
      }
    } catch (e) {
      &#x2F;&#x2F; Lookup failed; fall through to validation error handling
      console.log(&#x27;Lookup threw error:&#x27;, e);
    }
  } else {
    console.log(&#x27;Skipping lookup:&#x27;, { parsed_success: parsed.success, hasFlightNumber: !!body.flightNumber, hasFrom: !!body.from, hasTo: !!body.to });
  }

  if (!parsed.success) {
    if (lookupAttempted &amp;&amp; !lookupFound &amp;&amp; body.flightNumber) {
      if (lookupNoElapsedMatch) {
        return apiError(&#x27;Flight not yet arrived. Specify a departure date to look up past flights.&#x27;);
      }
      return apiError(&#x27;No matching flight route found for provided flight number&#x27;);
    }
    return json(
      { success: false, errors: parsed.error.errors },
      { status: 400 },
    );
  }
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

4) Rate limiting is real
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

✅ Saved (with route + times)
or ❌ Needs clarification (multiple matches)
Link references (keep these, replace with your real URLs)
AirTrail Save Flight docs:
https:&#x2F;&#x2F;airtrail.johan.ohly.dk&#x2F;docs&#x2F;api&#x2F;flight&#x2F;save-flight

AeroDataBox RapidAPI OpenAPI:
https:&#x2F;&#x2F;doc.aerodatabox.com&#x2F;docs&#x2F;openapi-rapidapi-v1.json
Prose
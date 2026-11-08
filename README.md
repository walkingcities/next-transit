# next-transit

Speech-ready next departures for Siri Shortcuts.

The Shortcut sends the phone’s latitude and longitude, plus what the person asked for. The server picks the nearest stop for that service and answers in minutes from now.

“Hey Siri, Next Transit.” Then: “Caltrain.” Or “two northbound Caltrain.” Or “bus 296.”

```json
{
  "stop": "Palo Alto",
  "service": "Caltrain",
  "directions": [
    {
      "direction": "southbound",
      "departures": [
        { "minutes": 7, "headsign": "San Jose Diridon" },
        { "minutes": 22, "headsign": "San Jose Diridon" }
      ]
    },
    {
      "direction": "northbound",
      "departures": [
        { "minutes": 4, "headsign": "San Francisco" },
        { "minutes": 19, "headsign": "San Francisco" }
      ]
    }
  ],
  "speech": "Southbound from Palo Alto: 7 minutes, then 22 minutes. Northbound: 4 minutes, then 19 minutes."
}
```

Times are minutes from now. The response does not include clock times.

This repository is the project home. The service is not implemented yet. It is meant to be hosted on Vercel, or a similar HTTPS host, so more than one person can use it.

## What the person has to say

The service name is required. `Caltrain`, or `bus 296`. With no service, the spoken answer is that we need one. A location alone is not enough, because the nearest stop might be a bus, a train, or nothing useful.

Direction is optional. North or south uses that direction. If they leave it out, the answer includes both.

Count is 1 or 2. One is the default. Two speaks both gaps: “7 minutes, then 22 minutes.”

## Shortcut

Apple Shortcuts can send the user’s location. The action is **Get Current Location**. It exposes **Latitude** and **Longitude**. Put those on the request to `POST /v1/next`. The first run asks for location permission for Shortcuts.

Siri starts a personal shortcut by its name. Words said in the same breath are not passed in. “Hey Siri, Next Transit two southbound Caltrain” does not deliver “two southbound Caltrain” to a shortcut named Next Transit. After the shortcut starts, **Ask for Input** (or **Dictate Text**) listens for the service, direction, and count. Send that string as `q`, with `lat` and `lng`.

```json
{ "lat": 37.443, "lng": -122.165, "q": "two southbound Caltrain" }
```

The Shortcut then reads `speech` and uses **Speak Text**.

On a HomePod, Personal Requests run the shortcut on the paired iPhone. **Get Current Location** is that iPhone’s GPS, not the speaker’s place in the room. The nearest stop is right when the phone is with the person. Test the shortcut from Siri, not only from the Shortcuts app. Location actions have failed under Siri even when they worked from the app.

A native app with App Intents could take “two southbound Caltrain” in the same sentence as the shortcut name. That is a later interface. The first one is a Shortcut plus `q`.

## Request rules

`q` is parsed on the server:

| `q` | Result |
| --- | --- |
| `Caltrain` | Nearest Caltrain stop. Next departure each direction. |
| `southbound Caltrain` | That direction only. One departure. |
| `two northbound Caltrain` | Two departures, minutes for both. |
| `bus 296` | Nearest stop served by route 296. Both directions if the route has them. |
| `next train` | Spoken error. No service was named. |

The stop is the nearest stop of that service to `lat`/`lng`. If nothing is close, say so. Headsigns can differ within one direction; the spoken line still leads with the minutes.

## Hosting

Vercel is the host. Each request is different because the stop depends on the phone’s coordinates, so the sentence cannot be cached. The 511 realtime feed can.

Vercel Hobby cron runs at most once a day, so a cron is the wrong way to refresh train positions. On each request, read a slim snapshot from [Vercel Runtime Cache](https://vercel.com/docs/caching/runtime-cache) with a TTL of about 30 seconds. If the entry is missing, fetch 511, reduce it to upcoming departures, and store that. A cache item is limited to 2 MB, so store the reduced departures, not the raw Bay Area protobuf. The 511 API key stays in Vercel environment variables.

Static stops (id, name, lat, lng, routes, direction) ship with the deployment. Finding the nearest stop is local math. Realtime minutes come from the cached snapshot.

Cold starts can add a second or two while Siri is waiting. That is acceptable for a shared Shortcut. The 511 key never appears in the Shortcut.

A public proxy of 511 data has to follow the [511 data agreement](https://511.org/sites/default/files/pdfs/511_Data_Agreement_Final.pdf).

## Why this is its own project

Several open-source systems already answer “what is departing.” None of them is a small HTTPS endpoint that takes a phone location and a short phrase and returns a sentence of relative minutes.

| Project | What it already does | Gap for this use |
| --- | --- | --- |
| [OneBusAway](https://github.com/OneBusAway/onebusaway-application-modules) and [Maglev](https://github.com/OneBusAway/maglev) | GTFS plus GTFS-RT, REST arrivals API, sign mode, Alexa, Twilio voice. Maglev can load the Bay Area 511 regional feed. | Arrivals JSON is large. A Shortcut would still parse it. |
| [transpo-rt](https://github.com/etalab/transpo-rt) | GTFS plus GTFS-RT exposed as SIRI Lite stop-monitoring JSON. | Response is the transit standard, not a spoken sentence. |
| [MOTIS](https://github.com/motis-project/motis) / [Transitous](https://transitous.org) | Trip planning over GTFS and realtime feeds. | Built for routes between places, not “next trip from the nearest stop.” |
| [curlbus](https://github.com/elad661/curlbus) | `curl` a stop and get the next buses, including JSON. | Israel Ministry of Transport only. |
| [NEXT-Transit](https://github.com/next-transit/next-transit-api) | Older multi-city GTFS API. The GitHub org `next-transit` is unrelated to this repo. | Schedule-oriented app API, last active years ago. |
| [siri-shortcut-hk-bus-eta](https://github.com/kennyfong19931/siri-shortcut-hk-bus-eta) | A Shortcut and site for Hong Kong bus and rail ETAs. | One region, and the parsing lives in the Shortcut. |

Bay Area realtime data is already public through [511 SF Bay](https://511.org/open-data/transit): GTFS, GTFS-RT, and SIRI-style stop monitoring. Caltrain is agency `CT`. The regional bundle is `RG`. A 511 API key is free and required.

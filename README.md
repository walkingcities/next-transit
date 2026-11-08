# next-transit

Speech-ready next departures for Siri Shortcuts.

Say “Hey Siri, Next Caltrain.” The Shortcut fetches one URL and speaks the sentence. The station and direction live in a preset, so Siri does not have to infer that you mean Palo Alto Caltrain.

```json
{
  "station": "Palo Alto",
  "direction": "southbound",
  "departures": [
    { "minutes": 7, "headsign": "San Jose Diridon" },
    { "minutes": 22, "headsign": "San Jose Diridon" }
  ],
  "speech": "The next southbound Caltrain leaves Palo Alto in 7 minutes. The following train leaves in 22 minutes."
}
```

The Shortcut is three actions: Get Contents of URL, Get Dictionary Value (`speech`), Speak Text. The same JSON can drive a front-door display.

This repository is the project home. The service is not implemented yet.

## Why this is its own project

Several open-source systems already answer “what is departing.” None of them is a small HTTPS endpoint whose whole job is a sentence a Shortcut can speak, with the agency key kept on the server.

| Project | What it already does | Gap for this use |
| --- | --- | --- |
| [OneBusAway](https://github.com/OneBusAway/onebusaway-application-modules) and [Maglev](https://github.com/OneBusAway/maglev) | GTFS plus GTFS-RT, REST arrivals API, sign mode, Alexa, Twilio voice. Maglev can load the Bay Area 511 regional feed. | Arrivals JSON is large. A Shortcut would still parse it. Running the full server is a bigger product than one spoken sentence. |
| [transpo-rt](https://github.com/etalab/transpo-rt) | GTFS plus GTFS-RT exposed as SIRI Lite stop-monitoring JSON. | Response is the transit standard, not a spoken sentence or a saved preset. |
| [MOTIS](https://github.com/motis-project/motis) / [Transitous](https://transitous.org) | Trip planning over GTFS and realtime feeds. | Built for routes between places, not “next train from my station.” |
| [curlbus](https://github.com/elad661/curlbus) | `curl` a stop and get the next buses, including JSON. | Israel Ministry of Transport only. |
| [NEXT-Transit](https://github.com/next-transit/next-transit-api) | Older multi-city GTFS API. The GitHub org `next-transit` is unrelated to this repo. | Schedule-oriented app API, last active years ago, not a Shortcut speech endpoint. |
| [siri-shortcut-hk-bus-eta](https://github.com/kennyfong19931/siri-shortcut-hk-bus-eta) | A Shortcut and site for Hong Kong bus and rail ETAs. | One region, and the parsing lives in the Shortcut. |

Bay Area realtime data is already public through [511 SF Bay](https://511.org/open-data/transit): GTFS, GTFS-RT, and SIRI-style stop monitoring. Caltrain is agency `CT`. The regional bundle is `RG`. A 511 API key is free and required. Palo Alto’s stop code still needs to be read from the 511 stops API before the first preset is hardcoded.

A public proxy of 511 data has to follow the [511 data agreement](https://511.org/sites/default/files/pdfs/511_Data_Agreement_Final.pdf). A personal instance that uses your own key is the straightforward way to start.

## Shortcut

1. In Shortcuts, create a shortcut named `Next Caltrain`.
2. Add **Get Contents of URL** for `https://<host>/v1/next?preset=palo-alto-caltrain-south`.
3. Add **Get Dictionary Value** for the key `speech`.
4. Add **Speak Text**.
5. On HomePod, turn on Personal Requests in the Home app so “Hey Siri, Next Caltrain” runs that shortcut through your iPhone.

Siri’s built-in transit answers can search, ask a clarifying question, or hand you Apple Maps directions. They do not reliably mean “Palo Alto, southbound, in minutes.” The shortcut name carries that.

## Hosting

The service needs public HTTPS, a place to store the 511 API key, and a short cache. 511 updates on its own cadence, about every 30 seconds. Every Siri request should read the cache, not call 511 again.

| Option | Good fit | What to watch |
| --- | --- | --- |
| **Cloudflare Workers**, cron every 30 seconds, cache in KV | The public Walking Cities instance. HTTPS and a custom domain are included. The worker returns the cached sentence. | GTFS-RT is protobuf. Parsing it in a Worker is fussier than in a normal process. SIRI StopMonitoring JSON from 511 avoids protobuf if that endpoint has the expected times you want. |
| **Raspberry Pi and Cloudflare Tunnel** | Your house, and the same machine as a front-door display. `cloudflared` gives HTTPS without opening a port. | The Pi has to stay on. Fine for you. A weak default for other people. |
| **Fly.io or any small VPS** | An always-on poller that holds GTFS-RT in memory and serves the speech JSON. A small machine handles the Bay Area feed. | You operate the box, TLS, and deploys. |
| **Vercel or another request-scoped function** | A thin reshape of one 511 JSON call. | Cold starts add delay while Siri is waiting. Without a shared cache, many shortcuts will share one 511 quota. |
| **Tailscale Funnel** | Only your own devices. | Not a service other people can add a shortcut for. |

Recommended split:

- **You, this month:** Pi plus Cloudflare Tunnel. One preset, Palo Alto southbound. The display and the Shortcut share that URL.
- **Anyone else:** a Cloudflare Worker in front of a cached feed, with presets in this repo (`palo-alto-caltrain-south`, and later other stations). Shortcuts stay free of API keys.

Preset config belongs in git. Secrets stay in the host’s environment. The URL a stranger puts in Shortcuts should not contain a 511 key.

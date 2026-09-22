# My Trips CTA Mapping

| Stage   | CTA(s) on My Trips | Opens |  
|---|---|---|
| `new`   | New trip | Home page |  
| `matching`   | Resume matching | Chat with the persisted active owner |
| `recommendation_ready` | Generate recommendations | Matching chat |
| `recommended`   | Review recommendations or Review match outcome | Ask-mapped comparison / terminal outcome |
| `matched`   | Review recommendations \| Want to plan? | Preserved comparison and selection \| Planner |
| `planning`  | Review recommendations, when available \| Resume planning | Recommendations \| UI planning placeholder |
| `planned`   | Open Dashboard | Trip Dashboard |
| `booked`    | Open Dashboard | Trip Dashboard |
| `done`      | View Dashboard | Trip Dashboard (read-only history) |
| Any stage with `itinerary_state.status == "ready"` | Open Dashboard (or View Dashboard once `done`) | Trip Dashboard — takes priority over the stage-only row above |

CTA labels and destinations are UI-owned. Opening recommendation review restores the stored comparison, selected option, and expanded recommendation state without changing lifecycle stage. Opening a trip restores `active_agent`; the stage label alone never reroutes a specialist continuation through Scout.

My Trips (TWM-108) also uses this table for its adaptive landing resolver: a trip whose `stage` is `new` with no `trip_context` yet is not a real trip for either surface and is excluded; a trip with `itinerary_state.status == "ready"` opens the Dashboard directly regardless of stage, matching the priority row above.

# My Trips CTA Mapping

| Stage   | CTA(s) on My Trips | Opens |  
|---|---|---|
| `new`   | New trip | Home page |  
| `matching`   | Resume matching | Chat with the persisted active owner |
| `recommendation_ready` | Generate recommendations | Matching chat |
| `recommended`   | Review recommendations or Review match outcome | Recommendations / terminal outcome |
| `matched`   | Review recommendations \| Want to plan? | Recommendations \| Planner |  
| `planning`  | Review recommendations, when available \| Resume planning | Recommendations \| UI planning placeholder |

CTA labels and destinations are UI-owned. Opening a trip restores `active_agent`; the stage label alone never reroutes a specialist continuation through Scout.

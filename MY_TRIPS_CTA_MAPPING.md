# My Trips CTA Mapping

| Stage   | CTA(s) on My Trips | Opens |  
|---|---|---|
| `new`   | New trip | Home page |  
| `matching`   | Resume matching | Chat with the persisted active owner |
| `recommendation_ready` | Generate recommendations | Matching chat |
| `recommended`   | Review recommendations or Review match outcome | Ask-mapped comparison / terminal outcome |
| `matched`   | Review recommendations \| Want to plan? | Preserved comparison and selection \| Planner |
| `planning`  | Review recommendations, when available \| Resume planning | Recommendations \| UI planning placeholder |

CTA labels and destinations are UI-owned. Opening recommendation review restores the stored comparison, selected option, and expanded recommendation state without changing lifecycle stage. Opening a trip restores `active_agent`; the stage label alone never reroutes a specialist continuation through Scout.

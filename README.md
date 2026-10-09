# Buyer extension

A Chrome extension I built and owned at Etainement, one of the larger ticket brokers in the country (2022 to 2026). Over a hundred buyers ran it inside Ticketmaster during on-sales. This repository is a write-up; the code is the company's. Full case study with the system map and drawings: [scottbarclay.dev/work/buyer-extension](https://scottbarclay.dev/work/buyer-extension).

![The extension painting a manager's rules onto a seat map (synthetic venue)](https://raw.githubusercontent.com/sbrcly/portfolio-v4/main/public/diagrams/buyer-extension-still.svg)

## The problem

A ticket broker buys tickets the moment they go on sale and resells them. On the morning of a big on-sale, a hundred buyers sit at Ticketmaster seat maps, each with a few seconds to pick seats before they are gone. A manager decides which seats are worth buying and at what price. Before this tool, those decisions reached buyers as Slack messages sent while the sale moved. Buyers missed them, or read them late, and carted the wrong seats. Every wrong cart cost money.

The obvious fix is to show the rules on the seat map itself. The difficulty is that the seat map belongs to Ticketmaster: we do not control its code, seat elements carry no usable ID, the map renders late and redraws on zoom, the markup changes month to month, and a Chrome extension under Manifest V3 cannot read the page's network responses.

## What it did

- **Painted the rules.** Managers wrote buy rules (sections, rows, maximum price, maximum tickets per event and per account) in the company's pricing portal. The extension fetched them and painted them onto the seat map the buyer already had open.
- **Recorded the purchase.** Event page load, waiting-room position, sign-in, cart, failed cart, checkout error, confirmation: each noticed on the page, stamped, and sent to BigQuery (kept for good) and Firestore (kept for an hour, read by the on-sale monitor).
- **Relayed verification codes.** When Ticketmaster asked for a one-time code, the extension fetched it from a relay service and put it on the page and the clipboard.
- **Worked the approval desk too.** In the third-party tool where managers approved carts, it shaded rows that matched a rule, showed each event's maximum as a gauge, and added a button that posted a buyer's bonus to Slack.

## How it worked

**Five JavaScript worlds.** Chrome keeps extensions in separate compartments, so a rule crossed five: the portal page (where the manager's login lives), the extension's background worker, the content script (inside the Ticketmaster tab, but sandboxed from its JavaScript), a script injected into Ticketmaster's own world (the only place seat IDs can be read, from the React internals on each element), and the identity frame on a third domain where codes are requested. Every hop is a message.

**The poll.** Every fifteen seconds while the tab is visible; stops when hidden; no repaint if nothing changed; pan and zoom repaint from cached rules one second after movement stops with no network call. No portal tab open means a red icon before the sale starts.

**The code relay.** An Express service holds the HTTP response open until the code exists. It checks a store first (a code from the last fifteen minutes is returned at once). For a phone line it opens that line's port on an SMS gateway and polls every ten seconds for up to ninety; a webhook writes arriving texts to Firestore keyed by line. Only one port per gateway group can be open at a time, so the service is also a scheduler: a port-state map, a queue per group, held responses for waiting callers, and release of the port on every exit path including failure. For an inbox, a Python mail hook on a Google Cloud VM runs once per email, parses each field in its own try block, and upserts the latest code per address into Postgres.

**Hard parts.** Matching rules to seats on a map we did not own (three generations: seat labels, grid coordinates, then React internals); reading network responses under Manifest V3 (a document-start wrapper around fetch and XMLHttpRequest with a buffer for early responses); timing on a late-rendering single-page app (observers that fire once, debounce, re-entrancy guard, pause when hidden); the cross-origin identity frame (window messages with explicit origins); auth without a login (the portal's token read from an open tab, one pinned extension ID on the backend); markup that changes (sixty test hooks, each spelled three ways); recording purchases nobody typed in (embedded page data first, DOM second, keyed per tab, with a popup for human correction).

**What I would do differently.** A guard for the day Ticketmaster's internals change shape; a test suite; one shared rules cache instead of one poll per tab; and the port scheduler's queues in a store instead of process memory.

Stack: TypeScript, React, Vite, Manifest V3, Express, Firestore, BigQuery, Postgres, Python.

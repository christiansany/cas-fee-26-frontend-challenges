# Frontend challenges and possible solutions

CAS Frontend Engineering at OST · Wednesday, 28 October 2026 · 17:15 to 21:50

Tonight is a menu. You hear a one-minute pitch for every topic, then you vote. We cover the topics with the most votes, in vote order, and there are more topics than time. Every topic stands on its own, so you can read the rest here later in any order.

Most topics take a problem you know from Project 2 and show what it looks like in a large application: many teams, many services, millions of users. The examples use the Lending Desk, the equipment lending service you know from an earlier evening, grown into a company-wide platform.

## The menu

| Topic | Duration | What it is about |
| --- | --- | --- |
| [Lazy loading with React](./topics/lazy-loading/README.md) | ~45 min | Load what the user is about to see, avoid layout shifts with skeletons, and prefetch before the click. |
| [Compound components with React](./topics/compound-components/README.md) | ~45 min | Split one big configuration object into parts you compose in JSX, like `<select>` and `<option>`. |
| [Responsive images](./topics/responsive-images/README.md) | ~90 min | Density and width descriptors, `sizes`, `<picture>` and modern formats, and calculating `sizes` in a grid. |
| [When only part of the data fails](./topics/partial-failures/README.md) | ~40 min | Sort data by impact, and build error boundaries that catch only the errors addressed to them. |
| [When the network lets you down](./topics/network-failures/README.md) | ~30 min | Time budgets, retries only where they are safe, cancellation, and error responses you can read. |
| [Logging in a large application](./topics/logging/README.md) | ~40 min | A logger with context, browser errors that reach a server, sampling and noise filters, and logs that leak data. |
| [Debugging production](./topics/debugging-production/README.md) | ~30 min | Follow one error from the browser through every service, with one ID, readable stack traces and clear ownership. |
| [GraphQL vs REST](./topics/graphql-vs-rest/README.md) | ~40 min | What GraphQL solves when many teams share a page, how sites move to one federated graph, and why REST stays. |
| [Who loads which data?](./topics/data-per-component/README.md) | ~40 min | Colocated fragments and data masking, and loading what can wait without a waterfall. |
| [Lists, filters and pagination](./topics/lists-and-pagination/README.md) | ~30 min | The URL as the single source of truth, offset vs cursor pagination, and a "load more" that feels instant. |
| [Keeping data consistent in the browser](./topics/client-cache/README.md) | ~30 min | Query caches and normalised caches, optimistic UI and when not to use it, and sync across tabs. |

## Archive

Topics from earlier years that are not on the menu right now: [archive](./archive/README.md).

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
| [When parts of the page fail](./topics/partial-failures/README.md) | ~40 min | Decide per piece of data what happens when it fails, one error boundary per section, and requests that fail predictably. |
| [Logging and debugging in production](./topics/logging/README.md) | ~40 min | Structured logs from server and browser, tied together by one ID that the user sees as a support code. |
| [GraphQL vs REST](./topics/graphql-vs-rest/README.md) | ~40 min | Ask for exactly the data a page needs, let each component declare its part, and know what that costs. |
| [Lists, filters and pagination](./topics/lists-and-pagination/README.md) | ~30 min | The URL as the single source of truth, keeping the old list visible while loading, and offset vs cursor pagination. |

## Archive

Topics from earlier years that are not on the menu right now: [archive](./archive/README.md).

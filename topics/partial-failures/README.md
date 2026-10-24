# When parts of the page fail

> A large page draws on many services, and on any given day one of them is slow or down. If one failure takes down the whole page, the page is only as reliable as its weakest service. We decide for every piece of data what happens when it fails, and give each section its own error boundary, so a failure costs one box, not the page. Then we make requests fail in a predictable way: a time budget per request, and retries only where they are safe.

**Duration:** about 40 minutes

## Contents

- [Where you are coming from](#where-you-are-coming-from)
- [What a partial response looks like](#what-a-partial-response-looks-like)
- [Sort the data by impact](#sort-the-data-by-impact)
- [One error boundary per section](#one-error-boundary-per-section)
- [From a failed field to a fallback](#from-a-failed-field-to-a-fallback)
- [Design the fallback](#design-the-fallback)
- [Make requests fail predictably](#make-requests-fail-predictably)
- [Takeaways](#takeaways)
- [Further reading](#further-reading)

## Where you are coming from

In Project 2 you wrap `fetch` in `try`/`catch` and show an error message when it fails. With one API behind your app, that is enough.

Now imagine the Lending Desk has grown into a company-wide platform with millions of users. Item catalogue, availability, reservations, reviews and recommendations each run as their own service, with its own team and deploys. The item page draws on about five of them, and on any given day one is slow or down. The goal of this topic: **when one piece of data fails, only the part of the page that needs it notices.**

```mermaid
flowchart LR
  subgraph before["Before: availability fails"]
    b1["Something went wrong"]
  end
  subgraph after["After: availability fails"]
    a1["Cordless drill, photos, owner"]
    a2["Availability can't be loaded. Retry"]
    a3["Reviews"]
    a4["Recommendations"]
  end
  before -- "one boundary per section" --> after

  classDef failed fill:#fde2e1,stroke:#c0392b,color:#000
  class b1,a2 failed
```

## What a partial response looks like

Large pages often load their data with GraphQL: one query names the fields the page needs, and a server collects them from the services:

```graphql
query ItemPage {
  item(id: "42") {
    name
    rating
    availability { status nextFreeDate }
  }
}
```

When the availability service times out, the server still answers: `availability` is `null`, the failure is listed under `errors` with its `path`, and the rest arrives.

```json
{
  "data": {
    "item": { "name": "Cordless drill", "rating": null, "availability": null }
  },
  "errors": [
    { "message": "Availability service timed out", "path": ["item", "availability"] }
  ]
}
```

The HTTP status is a 2xx, usually `200`, so `res.ok` is `true`. A REST endpoint that merges several services can follow the same pattern: return what it has, plus a list of what failed.

### `null` alone tells you nothing

`rating: null` means "nobody has rated this drill yet". `availability: null` means "the service failed". A component that only sees the value might show "Not available" for a drill that is free to borrow. So whatever reads a value must also check whether an error came with it.

Docs: [GraphQL spec, Handling Execution Errors](https://spec.graphql.org/September2025/#sec-Handling-Execution-Errors)

## Sort the data by impact

Before you write any error handling, decide for every piece of data what happens when it is missing. Three levels are enough:

- **Critical:** the page makes no sense without it. Show an error page.
- **Important:** the user needs it, but can still do something. Show a message and a retry.
- **Optional:** nice to have. Hide the section.

```mermaid
flowchart LR
  item["Name, photos, owner"] -- "fails" --> c["Critical: error page<br/>(keep this group small)"]
  avail["Availability, Reserve button"] -- "fails" --> i["Important: message and retry"]
  reviews["Reviews"] -- "fails" --> o["Optional: section hidden"]
  recs["Recommendations"] -- "fails" --> o

  classDef critical fill:#fde2e1,stroke:#c0392b,color:#000
  classDef important fill:#fff4d6,stroke:#b7791f,color:#000
  classDef optional fill:#e6f4ea,stroke:#2f855a,color:#000
  class item,c critical
  class avail,i important
  class reviews,recs,o optional
```

This is a product decision. Ask the product owner: "If availability is down, can a user still do something useful here?"

"Not found" is not "failed": a missing item is a 404 page, a failed catalogue service is an error page. Keep the two apart.

## One error boundary per section

An **error boundary** is a React component that catches errors thrown while rendering its children and shows a fallback instead. Error boundaries are still class components, so use the library `react-error-boundary` rather than writing one.

Each level from the diagram becomes a boundary. An error rises to the nearest boundary above it:

```tsx
import { ErrorBoundary } from "react-error-boundary";

export function ItemPage({ response }: { response: ItemPageResponse }) {
  return (
    <ErrorBoundary fallback={<ErrorPage />} onError={logFallback}>
      <ItemHeader response={response} /> {/* critical: reaches the page boundary */}
      <ErrorBoundary fallback={<AvailabilityUnknown />} onError={logFallback}>
        <Availability response={response} />
      </ErrorBoundary>
      {/* 👇 optional: a null fallback hides the section */}
      <ErrorBoundary fallback={null} onError={logFallback}>
        <Reviews response={response} />
      </ErrorBoundary>
    </ErrorBoundary>
  );
}
```

`onError` runs for every caught error; `logFallback` sends it to your error monitoring. For a retry button, use `fallbackRender`: it receives `resetErrorBoundary`, which renders the section again.

A boundary's position decides how much of the page disappears. Two things go wrong when many teams change a page:

- **A missing boundary.** A team adds a recommendations carousel without one. Its error reaches the page boundary, and an optional section takes down the page. A `Section` component that always includes a boundary makes it hard to forget.
- **Critical data inside a section.** If the owner's name fails inside the availability section, the user sees "Availability can't be loaded" next to an item without an owner. Render critical data outside every section boundary.

Docs: [React: Catching rendering errors with an error boundary](https://react.dev/reference/react/Component#catching-rendering-errors-with-an-error-boundary) · [`react-error-boundary`](https://github.com/bvaughn/react-error-boundary)

## From a failed field to a fallback

The boundary only helps if the component notices the failure. The page sends the query once and passes the `response` down. Each component checks `errors` for the path it reads:

```ts
type Path = readonly (string | number)[];
type GraphQLError = { message: string; path?: Path };

const startsWith = (path: Path, prefix: Path) => prefix.every((key, i) => path[i] === key);

// true if a field failed at, above or below `path`
export function failedAt(errors: readonly GraphQLError[] = [], path: Path): boolean {
  return errors.some(
    (error) => error.path !== undefined && (startsWith(error.path, path) || startsWith(path, error.path)),
  );
}

failedAt(errors, ["item", "availability"]); // true for ["item"], ["item", "availability"] or ["item", "availability", "status"]
failedAt(errors, ["item", "rating"]); // false if only availability failed
```

Now each section reacts in the way its level asks for:

```tsx
export function Availability({ response }: { response: ItemPageResponse }) {
  if (failedAt(response.errors, ["item", "availability"])) {
    throw new Error("Availability failed to load"); // 👇 the section's boundary shows its fallback
  }
  return <AvailabilityBadge availability={response.data.item.availability} />;
}

export function Reviews({ response }: { response: ItemPageResponse }) {
  if (failedAt(response.errors, ["item", "reviews"])) return null; // optional: hide it
  // ...
}
```

Throw when someone higher up must decide what to show. Return early when the component can decide on its own.

A boundary sees only errors thrown during render. A `fetch` that fails in `useEffect`, in a data library or in a click handler never reaches it unless you hand the error over: call `showBoundary` from `react-error-boundary`'s `useErrorBoundary`, or set `throwOnError: true` in TanStack Query.

```tsx
export function Recommendations({ itemId }: { itemId: string }) {
  const { showBoundary } = useErrorBoundary();
  const [items, setItems] = useState<Item[]>([]);
  useEffect(() => {
    getRecommendations(itemId).then(setItems).catch(showBoundary); // 👇 now the boundary sees it
  }, [itemId, showBoundary]);
  // ...
}
```

<details>
<summary>Going further: why <code>failedAt</code> looks above and below</summary>

If a non-null field fails, `null` bubbles up to the nearest parent that allows it. A failed `status!` can turn the whole `availability` object into `null` while the error's `path` points to `status`, below the path you read. A failed `item` nulls everything inside it, with a `path` above yours. Checking only the exact path misses both.

</details>

<details>
<summary>Going further: errors during server rendering</summary>

Error boundaries don't run while the page is rendered on the server. If a component throws inside a `<Suspense>` boundary, React sends that Suspense fallback in the HTML and tries the component again in the browser, where your error boundary gets its chance. A throw outside every `<Suspense>` boundary leaves no HTML to send, and your server must answer with an error page. So for sections rendered on the server, prefer the early return.

Docs: [React: Recovering from errors outside the shell](https://react.dev/reference/react-dom/server/renderToPipeableStream#recovering-from-errors-outside-the-shell)

</details>

<details>
<summary>Going further: Relay does the check for you</summary>

In Relay, Meta's GraphQL client, `@catch` on a field delivers an error as a value (`{ ok: false, errors }`), and `@throwOnFieldError` on a fragment throws to the nearest boundary when a component reads a failed field.

Docs: [Relay `@catch`](https://relay.dev/docs/guides/catch-directive/) · [Relay `@throwOnFieldError`](https://relay.dev/docs/guides/throw-on-field-error-directive/)

</details>

## Design the fallback

The boundary decides _whether_ the user sees a fallback. Design decides _what_ they see.

- **Hide optional sections.** Missing reviews beat an error box nobody can act on.
- **Say what is unknown.** "Availability can't be loaded right now", not "Not available", which is false.
- **Offer a retry for important data.** The cause is often temporary.
- **Keep the space.** A placeholder the size of the content keeps the "Reserve" button from jumping.
- **Report every fallback.** A hidden section looks fine to everyone, including you. The `onError` report shows when reviews disappear on 5% of page views.

Test every fallback: make one service fail with your chaos panel and look at the page.

## Make requests fail predictably

A fallback needs a failure. A request that hangs never fails, and a careless retry can book a drill twice.

### A time budget per request

`fetch` has no timeout of its own. If a service never answers, the spinner turns for minutes. Give every request a budget:

```ts
export async function getAvailability(itemId: string): Promise<Availability> {
  const res = await fetch(`/api/items/${itemId}/availability`, {
    signal: AbortSignal.timeout(3_000), // 👇 give up after 3 seconds
  });
  if (!res.ok) throw new Error(`HTTP ${res.status}`);
  return (await res.json()) as Availability;
}
```

When the time is up, `fetch` rejects with a `TimeoutError`. During server rendering, give optional data about a second, then render without it. In the browser, a spinner in one box can wait several seconds.

Docs: [`AbortSignal.timeout()`](https://developer.mozilla.org/en-US/docs/Web/API/AbortSignal/timeout_static)

### Retry only safe reads

Many failures are temporary: a service restarts, a connection drops. A read changes nothing: a `GET`, or a GraphQL query, even when sent with `POST`. Trying a read again often works, for the right failures:

| What happened                                 | Retry the read?                  |
| --------------------------------------------- | -------------------------------- |
| `408`, `429`, `502`, `503`, `504`             | Yes, after a growing pause       |
| `500`, or a network error (`TypeError`)       | At most once: it may be a bug    |
| Any other `4xx`, a timeout, a cancellation    | No: the answer won't change      |

Wait longer each time. Add a random offset so a thousand clients don't retry at the same moment:

```ts
const RETRY = new Set([408, 429, 502, 503, 504]);

export async function getWithRetry(url: string, attempts = 3): Promise<Response> {
  for (let attempt = 1; ; attempt++) {
    const res = await fetch(url, { signal: AbortSignal.timeout(3_000) });
    if (!RETRY.has(res.status) || attempt === attempts) return res;
    const pause = 200 * 2 ** (attempt - 1); // 200, then 400 ms
    await new Promise((resolve) => setTimeout(resolve, pause * (0.5 + Math.random())));
  }
}
```

Stop after two or three attempts. If the browser, a middle server and the backend each try three times, one click becomes 27 calls.

Docs: [`fetch()` exceptions](https://developer.mozilla.org/en-US/docs/Web/API/Window/fetch#exceptions) · [`Retry-After`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Retry-After)

### Never blind-retry a write

A write such as `POST /reservations` is different. The server can create the reservation, and then the connection drops before the answer arrives. The browser can't tell whether the write happened.

```mermaid
sequenceDiagram
  participant B as Browser
  participant R as Reservations service
  B->>R: POST /reservations
  Note over R: reservation created
  R--x B: connection drops
  B->>R: POST /reservations (retry)
  Note over R: second reservation
```

A blind retry gives the user two reservations for the same drill. Show an error instead, and let the user check their reservations. Data libraries agree: TanStack Query retries failed queries by default, but not mutations.

<details>
<summary>Going further: idempotency keys make writes safe to retry</summary>

The client creates a unique key once per user action and sends it with every attempt. The server stores the keys it has processed and answers a repeat with the original result instead of creating a second reservation.

```ts
const key = crypto.randomUUID(); // once per click, reused for every attempt
await fetch("/api/reservations", {
  method: "POST",
  headers: { "Idempotency-Key": key, "Content-Type": "application/json" },
  body: JSON.stringify({ itemId, from, to }),
});
```

`crypto.randomUUID()` works only on HTTPS or localhost. The `Idempotency-Key` header is an IETF draft that many payment and booking APIs already use.

Docs: [IETF draft: The Idempotency-Key HTTP Header Field](https://datatracker.ietf.org/doc/draft-ietf-httpapi-idempotency-key-header/) · [`crypto.randomUUID()`](https://developer.mozilla.org/en-US/docs/Web/API/Crypto/randomUUID)

</details>

<details>
<summary>Going further: cancel outdated requests</summary>

The user types "dr", then "drill". If the request for "dr" is slow, its results arrive last and overwrite those for "drill". Abort the previous request when the query changes:

```tsx
useEffect(() => {
  const controller = new AbortController();
  searchItems(query, controller.signal)
    .then(setResults)
    .catch((error: unknown) => {
      if (!controller.signal.aborted) setError(error); // 👇 a cancellation is not a failure
    });
  return () => controller.abort(); // runs before the next search starts
}, [query]);
```

Keep `AbortError` out of your error reports. Data libraries such as TanStack Query discard outdated answers for you.

Docs: [`useEffect`: fetching data](https://react.dev/reference/react/useEffect#fetching-data-with-effects) · [`AbortController`](https://developer.mozilla.org/en-US/docs/Web/API/AbortController)

</details>

## Takeaways

- A partial response delivers data and errors together with a success status. A `null` alone doesn't tell you which one you have.
- Sort every piece of data as critical, important or optional before you write the code.
- Give every important and optional section its own error boundary, keep critical data outside them, and report every fallback.
- A boundary only sees errors thrown during render. Hand it errors from effects and data libraries.
- Give every request a time budget, retry only safe reads, and never blind-retry a write.

## Further reading

- [React: Catching rendering errors with an error boundary](https://react.dev/reference/react/Component#catching-rendering-errors-with-an-error-boundary)
- [`react-error-boundary`](https://github.com/bvaughn/react-error-boundary)
- [GraphQL spec: Handling Execution Errors](https://spec.graphql.org/September2025/#sec-Handling-Execution-Errors)
- [GraphQL over HTTP: status codes](https://http-spec.graphql.org/draft/)
- [MDN: `AbortSignal`](https://developer.mozilla.org/en-US/docs/Web/API/AbortSignal)
- [MDN: HTTP response status codes](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status)
- Marc Brooker: [Exponential Backoff And Jitter](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/) (AWS Architecture Blog)
- [TanStack Query: Important defaults](https://tanstack.com/query/latest/docs/framework/react/guides/important-defaults)
- Related topic: [GraphQL vs REST, and why large sites use both](../graphql-vs-rest/README.md)

# GraphQL vs. REST und warum grosse Websites beides nutzen

> Öffne auf galaxus.ch eine Produkttyp-Seite. Die Navigation, die Produktliste und jede Produktkachel stammen von verschiedenen Teams und verschiedenen Services. Mit REST hangelt sich der Browser von einer API zur nächsten und bekommt gleichzeitig zu viele und zu wenige Daten. Mit GraphQL fragt die Seite in einem Request genau das an, was sie braucht, und jede Komponente deklariert ihren eigenen Teil. Wir schauen an, wie der Server eine solche Query beantwortet, was GraphQL kostet und warum grosse Websites beides betreiben.

**Dauer:** etwa 60 Minuten

## Inhalt

- [REST in Kürze](#rest-in-kürze)
- [Das Beispiel: eine Seite, drei Teams](#das-beispiel-eine-seite-drei-teams)
- [Die Seite mit REST laden](#die-seite-mit-rest-laden)
- [GraphQL kennenlernen](#graphql-kennenlernen)
- [Fragments: jede Komponente deklariert ihre Daten](#fragments-jede-komponente-deklariert-ihre-daten)
- [GraphQL oder REST?](#graphql-oder-rest)
- [Zusammenfassung](#zusammenfassung)

## REST in Kürze

Jede **Ressource** hat eine URL, und HTTP-Methoden sagen, was mit ihr geschehen soll.

```http
GET /api/products/5862971 HTTP/1.1
Accept: application/json
```

```json
{
  "id": 5862971,
  "name": "Banana plug SimpleLock",
  "brand": { "id": 4401, "href": "/api/brands/4401" }
}
```

Die Response ist JSON mit einem HTTP-Statuscode, hier `200`. Verwandte Ressourcen erscheinen oft als Links (`href`), denen ein Client mit dem nächsten Request folgen kann. `GET` liest, `POST` erstellt, `PUT` und `PATCH` ändern, `DELETE` löscht.

Gehören API und Frontend demselben Team, ist ein neues Feld auf einer Seite eine Änderung an einem Route Handler. **Das funktioniert nicht mehr, sobald die Seite Daten aus Services braucht, die anderen Teams gehören.**

## Das Beispiel: eine Seite, drei Teams

Unser Beispiel ist eine echte Seite: [Audio-Steckverbinder auf galaxus.ch](https://www.galaxus.ch/en/s1/producttype/audio-connectors-3137?tagIds=591-1388). Die Seite ist echt. Die Teams, APIs und Responses auf dieser Seite sind **erfunden**, um das Muster zu zeigen. Sie beschreiben nicht, wie Galaxus gebaut ist.

Stell dir vor, drei Teams teilen sich diese Seite:

- **Team A, Navigation**, gehört die Seite selbst und damit die Produkttypen: Titel, Breadcrumb und Sidebar.
- **Team B, Produktlisten**, gehört die Liste in der Seite: welche Produkte zu den Filtern passen, wie viele es sind und in welcher Reihenfolge.
- **Team C, Produkte**, gehören die Kacheln in der Liste: Name, Marke, Preis, Bilder und Bewertungen eines Produkts.

![Die Produkttyp-Seite von Team A, darin die Liste von Team B und darin die Kacheln von Team C](./assets/page-teams.png)

Jedes Team betreibt einen eigenen Service mit eigener REST API. **Die Verschachtelung ist auch die Ladereihenfolge:** Die Seite braucht die Produktliste von Team B, und die Liste braucht die Produkte von Team C.

**Die Kernidee dieses Themas:** Mit GraphQL holt diese Seite alle ihre Daten in einem Request, mit genau den Feldern, die sie anzeigt, und jede Komponente deklariert ihren eigenen Teil. Das kostet Dinge, die REST gratis mitbringt, deshalb nutzen grosse Websites beides.

## Die Seite mit REST laden

Wir laden die Seite mit REST, einen Service nach dem anderen.

### Runde 1: der Produkttyp von Team A

Der Browser kennt die Produkttyp-ID `3137` aus der URL. Ein Request an Team A liefert Titel, Breadcrumb und Sidebar:

```jsonc
// GET /api/product-types/3137
{
  "id": 3137,
  "name": "Audio connectors",
  "breadcrumb": [
    { "name": "IT + Multimedia", "href": "/api/categories/1" },
    { "name": "Audio", "href": "/api/categories/22" },
    { "name": "Audio Accessories + Spare Parts", "href": "/api/categories/316" }
  ],
  "navigation": [
    { "name": "Audio connectors", "href": "/api/product-types/3137", "active": true },
    { "name": "Audio player accessories", "href": "/api/product-types/2918" }
    // ... 12 more
  ],
  // 👇 the page shows none of these
  "seoText": "Explore our comprehensive selection of audio connectors ...",
  "synonyms": ["audio plug", "jack plug"],
  "createdAt": "2013-04-02T08:15:00Z",
  "links": {
    // 👇 where to get the products
    "products": "/api/product-lists?productTypeId=3137"
  }
}
```

### Runde 2: die Produktliste von Team B

Die Response des Produkttyps verlinkt auf die Produktliste. **Der Browser muss auf diese Response warten**, bevor er dem Link folgen kann. Er ergänzt den Filter aus der URL der Seite:

```jsonc
// GET /api/product-lists?productTypeId=3137&tagIds=591-1388&limit=48
{
  "totalCount": 1211,
  "filters": [
    { "id": "brand", "name": "Brand", "optionCount": 87 },
    { "id": "price", "name": "Price", "min": 0.95, "max": 289 }
    // ...
  ],
  "items": [
    // 👇 only links: the tiles need another round trip
    { "productId": 5862971, "href": "/api/products/5862971" },
    { "productId": 3376520, "href": "/api/products/3376520" }
    // ... 46 more
  ],
  "next": "/api/product-lists?productTypeId=3137&tagIds=591-1388&limit=48&offset=48"
}
```

### Runde 3: die Produkte von Team C

Jetzt weiss der Browser, welche 48 Produkte er zeigen soll, und fragt Team C **für jedes einzelne** an:

```jsonc
// GET /api/products/5862971
{
  "id": 5862971,
  "name": "Banana plug SimpleLock",
  "brand": { "id": 4401, "name": "clicktronic", "href": "/api/brands/4401" },
  "price": { "amount": 15.9, "currency": "CHF" },
  "images": ["/images/5862971-1.jpg", "/images/5862971-2.jpg" /* ... 10 more */],
  "rating": { "average": 4.6, "count": 70 },
  // 👇 the tile shows none of these
  "description": "The SimpleLock banana plug fits cables up to 6 mm² ...",
  "specifications": [{ "name": "Connector", "value": "4 mm banana" } /* ... 60 more */],
  "warranty": "2 years",
  "weightGrams": 12
}
```

### Under-Fetching, Over-Fetching und ein Wasserfall

Auf einer Zeitachse sieht man: **Die drei Runden warten aufeinander.**

![Der REST-Wasserfall: drei Runden von Requests, jede wartet auf die vorherige](./assets/rest-waterfall.png)

Drei Probleme zeigen sich, und die ersten beiden treten gleichzeitig auf:

- **Under-Fetching.** Einer Response fehlen Daten, die die Seite braucht, also muss der Browser einen weiteren Request machen. Die Produktliste enthält nur Links, also braucht jede Kachel einen zusätzlichen Request.
- **Over-Fetching.** Eine Response enthält Daten, die die Seite nicht braucht. Jedes Produkt kommt mit seiner Beschreibung und 60 Spezifikationen, obwohl die Kachel fünf Felder zeigt.
- **Wasserfall.** Jede Runde wartet auf die vorherige. `Promise.all` hilft nur innerhalb einer Runde: Die Kacheln warten trotzdem auf die Liste, und die Liste wartet auf den Produkttyp.

Auf einem Smartphone kann jeder Round Trip 150 ms oder mehr dauern, und 48 Produkte mit je 60 Spezifikationen sind viele Bytes für fünf Felder pro Kachel.

## GraphQL kennenlernen

### Schema, Query und Response

graphql.org stellt GraphQL mit drei Snippets vor: einem Schema, einer Query und einer Response. Hier sind sie, leicht angepasst.

**Das Schema** beschreibt jeden Typ und jedes Feld, das die API anbietet, wie ein TypeScript-Interface für die ganze API. `type Query` listet die Einstiegspunkte zum Lesen, wie deine `GET`-Routen.

```graphql
type Query {
  project(name: String!): Project
}

type Project {
  name: String
  tagline: String
  contributors: [User]
}
```

**Die Query** nennt die Felder, die der Client will. Diese hier fragt ein Feld eines Projekts an:

```graphql
{
  project(name: "GraphQL") {
    tagline
  }
}
```

**Die Response** hat dieselbe Form wie die Query, steckt in `data` und enthält nur die angefragten Felder:

```json
{
  "data": {
    "project": {
      "tagline": "A query language for APIs"
    }
  }
}
```

Das ist die ganze Idee: **Der Client bestimmt, welche Felder zurückkommen.** Ein paar weitere Begriffe, übersetzt in REST:

| GraphQL | In REST hiesse das | Bedeutung |
| --- | --- | --- |
| **Schema** | Ein OpenAPI-Dokument | Jeder Typ und jedes Feld, das die API anbietet |
| **Typ**, etwa `Project` | Die Form eines JSON-Bodys | Eine benannte Form |
| **Feld auf `Query`** | Ein Endpoint, `GET /projects/:name` | Ein Einstiegspunkt zum Lesen |
| **Query** | Ein `GET`-Request | Ein Lesezugriff, der die gewünschten Felder auflistet |
| **Mutation** | `POST`, `PUT`, `DELETE` | Ein Schreibzugriff, der ebenfalls die angefragten Felder zurückgibt |
| **Resolver** | Ein Route Handler | Die Serverfunktion hinter einem Feld |

**Jede Query und jede Mutation geht an dieselbe URL**, meist als `POST /graphql` mit der Query und ihren Variablen im JSON-Body. `!` bedeutet im Schema "nie null", `[User]` ist eine Liste.

Doku: [graphql.org](https://graphql.org/), [Queries](https://graphql.org/learn/queries/), [Schemas und Typen](https://graphql.org/learn/schema/), [Auslieferung über HTTP](https://graphql.org/learn/serving-over-http/)

### Die Produkttyp-Seite als eine Query

Jetzt zu unserer Seite. Der GraphQL-Server bietet dieses Schema an, vereinfacht:

```graphql
type Query {
  productType(id: ID!): ProductType
}

type ProductType {
  name: String!
  breadcrumb: [Category!]!
  navigation: [NavigationEntry!]!
  products(tagIds: [ID!], first: Int!): ProductList!
}

type ProductList {
  totalCount: Int!
  items: [Product!]
}

type Product {
  id: ID!
  name: String!
  brand: Brand!
  price: Money!
  imageUrl: String!
  rating: Rating
}

# Category, NavigationEntry, Brand, Money and Rating are left out
```

**Das Schema versteckt die Services.** `ProductType.products` sieht aus wie ein gewöhnliches Feld, obwohl der Service von Team B es beantwortet.

Die Seite schickt **eine Query mit allem, was sie anzeigt**. `$id` und `$tagIds` sind **Variablen**: Ihre Werte reisen als JSON neben der Query mit, wie Routenparameter.

<table>
<tr><th>Query</th><th>Response</th></tr>
<tr>
<td>

```graphql
query ProductTypePage($id: ID!, $tagIds: [ID!]) {
  productType(id: $id) {
    name
    breadcrumb { name }
    navigation { name url }
    products(tagIds: $tagIds, first: 48) {
      totalCount
      items {
        name
        brand { name }
        price { amount currency }
        imageUrl
        rating { average count }
      }
    }
  }
}
```

</td>
<td>

```jsonc
{
  "data": {
    "productType": {
      "name": "Audio connectors",
      "breadcrumb": [{ "name": "IT + Multimedia" } /* ... */],
      "navigation": [{ "name": "Audio player accessories", "url": "/en/s1/producttype/2918" } /* ... */],
      "products": {
        "totalCount": 1211,
        "items": [
          {
            "name": "Banana plug SimpleLock",
            "brand": { "name": "clicktronic" },
            "price": { "amount": 15.9, "currency": "CHF" },
            "imageUrl": "/images/5862971-1.jpg",
            "rating": { "average": 4.6, "count": 70 }
          }
          // ... 47 more
        ]
      }
    }
  }
}
```

</td>
</tr>
</table>

Ein Request, kein `seoText`, keine Spezifikationen, keine Links zum Folgen. **Für den Browser sind Over-Fetching und Under-Fetching weg.**

### Wenn ein Service ausfällt

**Fällt ein Service aus, kommt der Rest trotzdem an.** Das gilt für Felder, die `null` sein dürfen: Ein Fehler in einem Feld mit `!` setzt das nächste nullable Feld darüber auf `null`. Deshalb ist `items` im Schema ohne `!` deklariert. Angenommen, Team C ist down: `items` ist `null`, und ein Eintrag in `errors` sagt, wo.

```json
{
  "data": { "productType": { "name": "Audio connectors", "products": { "totalCount": 1211, "items": null } } },
  "errors": [{ "message": "Products unavailable", "path": ["productType", "products", "items"] }]
}
```

Der HTTP-Statuscode ist hier trotzdem `200`, also **prüf `errors` selbst**.

Aus der Praxis: Der GraphQL-Layer selbst steht selten im Weg, aber **die Fehlerbehandlung wird aufwändiger**. Alles kann klappen, alles kann schiefgehen, und dazwischen liegen viele mögliche Teilantworten. Für jede davon braucht die Seite ein sinnvolles Verhalten.

Doku: [Variablen](https://graphql.org/learn/queries/#variables), [Response](https://graphql.org/learn/response/)

### Resolver: wie der Server die Daten sammelt

Der Browser hat einen Request geschickt. Der GraphQL-Server muss die Daten trotzdem aus drei Services zusammentragen. Das macht er mit **Resolvern**: Funktionen, die wissen, wie sie den Wert eines Felds bekommen.

**Nicht jedes Feld braucht einen.** Gibt ein Resolver ein Objekt zurück, werden die Felder dieses Objekts direkt daraus gelesen. Ein eigener Resolver ist nur nötig, wenn die Daten eines Felds von woanders kommen oder Arbeit brauchen:

![Die Query der Seite: Vier Felder haben einen eigenen Resolver, der Rest kommt mit dem `parent`-Objekt](./assets/resolvers.png)

Im Code sehen die Resolver für unsere Seite so aus. Jeder Resolver bekommt das Ergebnis des übergeordneten Resolvers (`parent`), die Argumente des Felds (`args`) und einen `context` mit den API-Clients für diesen Request:

```ts
const resolvers = {
  Query: {
    // 👇 Team A: one call returns name, breadcrumb and navigation as well
    productType: (_parent, args, context) => context.teamA.getProductType(args.id),
  },
  ProductType: {
    products: (productType, args, context) =>
      context.teamB.getProductList({ productTypeId: productType.id, tagIds: args.tagIds, limit: args.first }),
  },
  ProductList: {
    // 👇 Team B returns ids only, so fetch all 48 products from Team C in one call
    items: (list, _args, context) => context.teamC.getProducts(list.productIds),
  },
  Product: {
    // a resolver that calls nothing: the schema offers one image, the service returns a list
    imageUrl: (product) => product.images[0],
  },
};
```

`name`, `breadcrumb`, `navigation`, `totalCount`, `price`, `brand` und `rating` haben keinen Resolver. GraphQL liest `productType.name` oder `product.price` aus dem `parent`-Objekt. Deshalb ist `getProductType` **ein Aufruf, nicht ein Aufruf pro Feld**.

Der Server führt die Resolver Ebene für Ebene aus: `products` wartet auf `productType`, und `items` wartet auf `products`. **Der Wasserfall ist noch da, aber er läuft direkt neben den Services:**

![GraphQL: ein Round Trip für den Browser, drei kurze Schritte auf dem Server](./assets/graphql-waterfall.png)

Auch das Over-Fetching ist umgezogen: Der Service von Team C liefert weiterhin das ganze Produkt mit 60 Spezifikationen, aber an den GraphQL-Server im selben Rechenzentrum. **Nur die Felder, die die Seite angefragt hat, reisen zum Smartphone**: fünf pro Kachel statt des ganzen Produkts.

Doku: [graphql.org: Execution](https://graphql.org/learn/execution/), [Apollo Server: Resolver](https://www.apollographql.com/docs/apollo-server/data/resolvers)

<details>
<summary>Weiterführend: das N+1-Problem und DataLoader</summary>

Unser `items`-Resolver holt alle 48 Produkte in einem Aufruf, weil er die ganze Liste der IDs auf einmal bekommt. Oft sieht es anders aus. Angenommen, `brand` käme aus einem eigenen Service und hätte deshalb einen Resolver: Ein `Product.brand`-Resolver läuft einmal pro Produkt, also verursachen 48 Produkte einen Request für die Liste plus 48 für die Marken. Das ist das **N+1-Problem**.

**DataLoader** sammelt die Keys, nach denen Resolver während der Ausführung der Query fragen, und ruft eine Batch-Funktion einmal auf:

```ts
// one loader per request, so cached brands never leak between requests
export function createBrandLoader(teamC: TeamCApi) {
  return new DataLoader<number, Brand>(async (ids) => {
    // 👇 one call for all ids
    const brands = await teamC.getBrands(ids);
    return ids.map((id) => brands.find((brand) => brand.id === id) ?? new Error(`No brand ${id}`));
  });
}

// resolver for Product.brand, with the loader in the context
const brand = (product: Product, _args: unknown, context: Context) => context.brandLoader.load(product.brandId);
```

Die Batch-Funktion muss pro Key ein Resultat zurückgeben, in derselben Reihenfolge wie die Keys.

Doku: [graphql.org: Performance](https://graphql.org/learn/performance/), [DataLoader](https://github.com/graphql/dataloader)

</details>

## Fragments: jede Komponente deklariert ihre Daten

Eine Query pro Seite löst den Wasserfall für den Browser. **Aber wer schreibt sie?** Die Seite besteht aus Komponenten, und die Komponenten gehören den drei Teams:

![Die Seite aufgeteilt in Komponenten: ProductTypePage umschliesst CategoryNavigation, ProductTypeHeader und ProductList, und ProductList rendert ProductTile](./assets/page-components.png)

- Schreibt **die Seite** die Query, listet sie jedes Feld jedes Teams auf. Jedes neue Feld auf einer Kachel heisst, die Query der Seite anzupassen, und niemand wagt es, ein Feld zu entfernen, also wächst die Query nur.
- Holt **jede Komponente** ihre eigenen Daten, sind die Teams wieder unabhängig, aber wir sind zurück bei vielen Requests und Wasserfällen.

Die Lösung heisst **Colocation** (hier nicht im Sinn von Server-Housing im Rechenzentrum): Jede Komponente deklariert ihre Felder direkt neben ihrem Code, als **Fragment**. Ein Fragment ist eine benannte Liste von Feldern auf einem Typ, und `...Name` fügt es in eine Query ein. Ein Name wie `ProductTile_product` bedeutet: was `ProductTile` von einem `product` braucht.

### Die Produktkachel und ihr Fragment

Die Kachel zeigt fünf Dinge, also listet ihr Fragment fünf Felder:

![Die Produktkachel und die Felder ihres Fragments](./assets/product-tile-fragment.png)

Queries und Fragments sind gewöhnliche Strings, deshalb **kann TypeScript sie nicht prüfen**. Ein Codegenerator wie GraphQL Codegen liest sie zusammen mit dem Schema und schreibt für jede einen Typ, so wie OpenAPI-Codegen das für REST tut. Du packst jeden String in die generierte Funktion `graphql(...)`, um diesen Typ zu bekommen. Die Funktion zum Demaskieren heisst im Client Preset standardmässig `useFragment`, obwohl sie kein React Hook ist. Wir benennen sie mit der Option `fragmentMasking: { unmaskFunctionName: 'getFragmentData' }` um.

```tsx
import { graphql, type FragmentType, getFragmentData } from "./gql";

export const ProductTileFragment = graphql(`
  fragment ProductTile_product on Product {
    name
    brand { name }
    price { amount currency }
    imageUrl
    rating { average count }
  }
`);

type ProductTileProps = { product: FragmentType<typeof ProductTileFragment> };

export function ProductTile(props: ProductTileProps) {
  // 👇 unmask: only the fields of ProductTile_product
  const product = getFragmentData(ProductTileFragment, props.product);
  return (
    <article>
      <img src={product.imageUrl} alt="" />
      <p>{product.price.amount.toFixed(2)}</p>
      <h3><strong>{product.brand.name}</strong> {product.name}</h3>
      <Rating average={product.rating?.average} count={product.rating?.count} />
    </article>
  );
}
```

### Die Query der Seite zusammensetzen

`ProductList` gehört die Liste, also fragt ihr Fragment die Anzahl an und setzt das Fragment der Kachel in jedes Item ein:

```tsx
export const ProductListFragment = graphql(`
  fragment ProductList_productType on ProductType {
    products(tagIds: $tagIds, first: 48) {
      totalCount
      items {
        id
        ...ProductTile_product
      }
    }
  }
`);

// in ProductList, after unmasking its own fragment
<h2>{productType.products.totalCount} products</h2>
{productType.products.items?.map((item) => <ProductTile key={item.id} product={item} />)}
```

`ProductTypePage`, die Seitenkomponente von Team A, setzt nur die Fragments ihrer direkten Kinder zusammen:

```graphql
query ProductTypePage($id: ID!, $tagIds: [ID!]) {
  productType(id: $id) {
    ...ProductTypeHeader_productType
    ...CategoryNavigation_productType
    ...ProductList_productType
  }
}
```

Die Fragments werden Teil der Query, also **schickt die Seite weiterhin einen Request**. Fügt Team C der Kachel ein Feld hinzu, ändert es `ProductTile` und dessen Fragment. **Die Query der Seite bleibt, wie sie ist.**

### Data Masking

**Data Masking** hält die Teams auseinander. `props.product` kommt **maskiert** an: Sein Typ versteckt jedes Feld, bis die Komponente es mit ihrem eigenen Fragment demaskiert. In `ProductList` heisst das:

```tsx
item.id; // fine: ProductList asked for it
item.price; // TypeScript error: only ProductTile asked for it
```

Kein Team hängt aus Versehen von den Feldern eines anderen Teams ab, und **wenn du ein Feld aus deinem Fragment entfernst, kann das nur deine eigene Komponente kaputt machen**.

Doku: [graphql.org: Fragments](https://graphql.org/learn/queries/#fragments), [The Guild: Fragments mit GraphQL Codegen](https://the-guild.dev/blog/unleash-the-power-of-fragments-with-graphql-codegen), [Codegen Client Preset](https://the-guild.dev/graphql/codegen/plugins/presets/preset-client)

<details>
<summary>Weiterführend: Relay</summary>

**Relay**, der GraphQL-Client von Meta für React, hat dieses Modell eingeführt. Sein Compiler generiert die Typen und maskiert die Daten auch zur Laufzeit, nicht nur in TypeScript. Eine Komponente liest ihr Fragment mit dem Hook `useFragment`. Die ESLint-Regel `relay/unused-fields` von Relay meldet Felder, die eine Komponente anfragt, aber nie liest.

Doku: [Relay: Thinking in Relay](https://relay.dev/docs/principles-and-architecture/thinking-in-relay/), [Relay: Fragments](https://relay.dev/docs/guided-tour/rendering/fragments/), [eslint-plugin-relay](https://github.com/relayjs/eslint-plugin-relay)

</details>

## GraphQL oder REST?

### Die Antwort von REST auf Under-Fetching: `?include=`

REST kennt ein Muster gegen Under-Fetching: Der Client verlangt, dass eine Response verwandte Ressourcen einbettet.

```http
GET /api/product-lists?productTypeId=3137&include=products
```

JSON:API nennt das **Inclusion of Related Resources**, Stripe nennt es **Expanding** einer Response (`expand[]=...`). Das löst das Problem nur teilweise:

- **Over-Fetching bleibt meist.** Ohne Feldauswahl kommt jedes eingebettete Produkt weiterhin mit seiner Beschreibung und 60 Spezifikationen. JSON:API kann das mit Sparse Fieldsets (`fields[...]`) eingrenzen, aber auch das muss jeder Endpoint selbst umsetzen.
- **Die Daten müssen dort verknüpfbar sein.** Der Service von Team B kann Produkte nur einbetten, wenn er an die Daten von Team C herankommt. Kann er das nicht, funktioniert das Muster gar nicht.
- **Jeder Endpoint implementiert es neu.** Jede Response, die Produkte einbettet, und tiefere Daten wie `include=products.brand`, muss die Verknüpfung selbst machen.

Ein Beispiel aus der Praxis: Bei den Kommentaren eines Blogposts haben wir die Antworten auf jeden Kommentar per `include` mitgeladen, also eine verschachtelte Pagination in einem Call. Die Benutzerdaten zu den Kommentaren kamen aber nicht mit, dafür brauchte es einen weiteren API-Call pro Kommentar. **Der Gewinn war bescheiden.**

**GraphQL macht das einmal, in den Resolvern, und jede Query kann es nutzen.**

Doku: [JSON:API: Inclusion of Related Resources](https://jsonapi.org/format/#fetching-includes), [Stripe: Expanding Responses](https://docs.stripe.com/expand)

### Was GraphQL kostet und wo REST weiterhin gewinnt

GraphQL verschiebt Arbeit vom Client auf den Server und ins Tooling:

| Thema | REST | GraphQL |
| --- | --- | --- |
| **Caching** | Die URL ist der Cache-Key. Browser und CDNs (geteilte Caches nahe bei den Nutzenden) speichern einen `GET` mit `Cache-Control`. | Meist ein `POST` an eine URL, den Caches überspringen. Eine **Persisted Query** hilft: ein `GET` mit einer kurzen ID für eine Query, die der Server schon kennt. |
| **HTTP-Features** | Redirects und Datei-Uploads sind gewöhnliches HTTP. | Nicht Teil von GraphQL. Dateien laufen meist über gewöhnliches HTTP. |
| **Fehler** | Ein Statuscode pro Request. | Ein fehlgeschlagenes Feld liefert meist trotzdem `200`, solange `data` Daten enthält. Monitoring, das Statuscodes zählt, verpasst es. |
| **Autorisierung** | Wird in jedem Route Handler geprüft. | Viele Queries erreichen dasselbe Feld, also prüfst du Berechtigungen einmal, im Code, den die Resolver aufrufen. |
| **Teure Queries** | Jeder Request ist im Voraus bekannt. | Clients schreiben ihre eigenen: 48 Produkte mit je 60 Spezifikationen und 50 Reviews. Server begrenzen Tiefe und Kosten. |
| **Betrieb** | `fetch` und Express. | Ein Schema, ein Server, den ein Team betreibt, Codegen, Schema-Checks im CI. |

Doku: [Best Practices](https://graphql.org/learn/best-practices/), [Autorisierung](https://graphql.org/learn/authorization/), [Sicherheit](https://graphql.org/learn/security/)

<details>
<summary>Weiterführend: Caching mit Persisted Queries</summary>

Queries dürfen mit `GET` geschickt werden, und das können Caches speichern. Lange Queries passen nicht in die Längenlimits von URLs, deshalb schickt der Client statt des Query-Texts eine kurze Kennung, eine **Persisted Query**: `GET /graphql?documentId=<hash>&variables=...` (die Parameternamen unterscheiden sich je nach Tool). Der Server muss trotzdem `Cache-Control` senden, und eine Response, die öffentliche Produktdaten mit Daten der eingeloggten Person mischt, darf nicht öffentlich gecacht werden.

Doku: [Auslieferung über HTTP](https://graphql.org/learn/serving-over-http/), [Relay: Persisted Queries](https://relay.dev/docs/guides/persisted-queries/)

</details>

### Beides in einer grossen Firma

Was nutzt also ein grosser Shop? **Beides, jeweils dort, wo es passt.** Im Hintergrund betreibt jedes Team eigene Services mit eigener Datenbank, untereinander meist mit REST APIs verbunden. Davor sitzen zwei Arten von Einstiegspunkten:

![Ein GraphQL-Layer für die Daten der Seiten, REST-Endpoints für alles andere, dahinter die Services der Teams](./assets/large-company.png)

- **Ein GraphQL-Layer** für die Seiten des Webshops und der Mobile App. Beide Clients fragen dasselbe Schema nach genau dem, was jeder Screen braucht.
- **REST-Endpoints** für alles, was gewöhnliches HTTP ist oder mit der Aussenwelt spricht: Login mit seinen Redirects, Datei-Uploads, Webhooks eines Payment-Providers und eine öffentliche API für Partnershops, die stabile URLs und eine Dokumentation in OpenAPI erwarten.

Ein einzelner GraphQL-Server, der 30 Services aufruft, **wird selbst zum Flaschenhals**: Jedes neue Feld braucht das Team, das ihn betreibt. Grosse Firmen teilen den Graphen deshalb oft auf, sodass jedes Team seinen Teil des Schemas betreibt und ein Gateway die Teile zu einem zusammenfügt. Das nennt sich **Federation**.

| Situation | Einsatz |
| --- | --- |
| Ein Team, eine App, eine Handvoll Ressourcen | REST |
| Öffentliche Daten, die ein CDN cachen soll, Datei-Uploads, Login-Redirects, Webhooks, Partner-APIs | REST |
| Eine Seite oder mehrere Clients, die Daten aus vielen Services kombinieren | GraphQL |
| Viele Teams, die Teile derselben Seite bauen | GraphQL mit Fragments |

Doku: [graphql.org: Federation](https://graphql.org/learn/federation/)

## Zusammenfassung

- Mit REST lädt der Browser die Seite **in einem Wasserfall** und bekommt gleichzeitig zu viele und zu wenige Daten.
- `?include=` hilft nur teilweise: Over-Fetching bleibt meist, die Daten müssen im Service verknüpfbar sein, und jeder Endpoint setzt es neu um.
- **GraphQL verknüpft die Daten auf dem Server:** Die Seite fragt in einem Request genau ihre Felder an, Resolver holen sie bei den Services der Teams.
- Den Wasserfall gibt es weiterhin, aber er läuft im Rechenzentrum.
- Mit Fragments deklariert jede Komponente ihre Felder, Data Masking hält die Teams unabhängig.
- GraphQL kostet, was REST gratis mitbringt: Caching über die URL, HTTP-Features, ehrliche Statuscodes. Dazu kommen aufwändigere Fehlerbehandlung bei Teilantworten und ein Server, den jemand betreiben muss.
- **Grosse Websites nutzen beides:** GraphQL für die Daten hinter ihren Seiten, REST an den Rändern.

# Compound Components mit React

**Das Problem:** Ein grosses Konfigurationsobjekt macht eine Komponente schwer zu lesen und schwer zu ändern.

**Dauer:** etwa 45 Minuten

## Inhalt

- [Was sind Compound Components?](#was-sind-compound-components)
- [Das Problem im Detail](#das-problem-im-detail)
- [Die Lösung](#die-lösung)
- [Beispiel: Accordion](#beispiel-accordion)

## Was sind Compound Components?

Compound Components sind ein Pattern, das den **State** und das **Verhalten** einer Gruppe von Komponenten kapselt, die Kontrolle über das Rendering der einzelnen Teile aber dir überlässt.

Das Ziel ist eine ausdrucksstärkere und flexiblere API für die Kommunikation zwischen der übergeordneten Komponente und ihren Kindkomponenten.

Ein gutes Beispiel ist `<select>` in Kombination mit `<option>` in HTML:

```html
<select>
  <option value="value1">name1</option>
  <option value="value2">name2</option>
  <option value="value3">name3</option>
</select>
```

Die beiden Elemente kommunizieren im Hintergrund miteinander. Das `<select>` hält den State, die `<option>`s geben vor, was ausgewählt werden kann. Das ermöglicht ein expressives Markup, und du musst die Optionen nicht über ein Attribut mitgeben.

```html
<!-- With an attribute 😱 -->
<select options="name1:value1;name2:value2;name3:value3"></select>
```

## Das Problem im Detail

Angenommen, wir müssen ein Menu bauen. Das Menu hat einen Button, der es öffnet und schliesst. Es enthält Links und Buttons. Ein Klick darauf soll die gewünschte Aktion auslösen und das Menu wieder schliessen.

Schauen wir uns verschiedene Arten der Implementierung an.

### Konfiguration

```jsx
function Example() {
  const menuConfig = {
    menuButton: {
      text: "Menu"
    },
    menuList: [
      {
        kind: "item",
        text: "Download",
        onClick: () => {
          /* ... */
        }
      },
      {
        kind: "link",
        text: "Show details",
        href: "/details"
      }
    ],
    closeOnItemClick: true
  };

  return <Menu {...menuConfig} />;
}
```

In diesem Beispiel erstellen wir ein kompliziertes Konfigurationsobjekt. Es enthält bereits alle Informationen, die jede Komponente innerhalb von `<Menu>` für Rendering und Verhalten braucht. Dieses Pattern ist weit verbreitet.

Ob das Menu offen oder geschlossen ist, verwaltet die `<Menu>`-Komponente selbst, und die Kindkomponenten können den State von `<Menu>` verändern.

#### Vorteile

1. `<Menu>` verwaltet seinen internen State selbst, also ob es offen oder geschlossen ist

#### Nachteile

1. `<Menu>` ist direkt für das Rendering aller Kindkomponenten verantwortlich, was die Komponenten sehr eng miteinander koppelt
2. Die `props` werden durch mehrere Komponenten durchgereicht. Das nennt man "Prop Drilling" (Props werden Ebene für Ebene weitergereicht), und das wird schnell umständlich und unübersichtlich
3. Wenn die Daten in einem anderen Format daherkommen, musst du sie zuerst auf die Struktur der Config mappen

```jsx
function Example() {
  // Data that comes from an API or is passed in as props
  const links = [
    { title: "Show details", target: "/details" },
    { title: "Show profile", target: "/profile" }
  ];

  const menuConfig = {
    menuButton: {
      text: "Menu"
    },
    menuList: [
      {
        kind: "item",
        text: "Download",
        onClick: () => {
          /* ... */
        }
      }
    ].concat(
      links.map((link) => ({
        // Data has to be mapped to the structure of the config
        kind: "link",
        text: link.title,
        href: link.target
      }))
    ),
    closeOnItemClick: true
  };

  return <Menu {...menuConfig} />;
}
```

### State selbst verwalten

```jsx
function Example() {
  const [isOpen, setIsOpen] = useState(false);

  const open = () => {
    setIsOpen(true);
  };

  const close = () => {
    setIsOpen(false);
  };

  const download = () => {
    // Download stuff ...
    close();
  };

  // Data that comes from an API or is passed in as props
  const links = [
    { title: "Show details", target: "/details" },
    { title: "Show profile", target: "/profile" }
  ];

  return (
    <Menu open={isOpen}>
      <MenuButton onClick={open}>Menu</MenuButton>
      <MenuList>
        <MenuItem onClick={download}>Download</MenuItem>
        {links.map((link) => (
          <MenuLink key={link.target} to={link.target} onClick={close}>
            {link.title}
          </MenuLink>
        ))}
      </MenuList>
    </Menu>
  );
}
```

In diesem Beispiel müssen wir kein kompliziertes Konfigurationsobjekt erstellen. Da die Komponenten jedoch nicht miteinander verbunden sind, müssen wir den State des Menus selbst verwalten.

#### Vorteile

1. Die Komponenten sind nicht mehr eng gekoppelt, und `<Menu>` muss sich nicht mehr um das Rendering seiner Kindkomponenten kümmern
2. Die Datenstruktur spielt keine Rolle mehr

#### Nachteile

1. Du musst den State selbst verwalten. Je nach Komponente wird das sehr kompliziert

## Die Lösung

Das Compound Components Pattern verbindet beide Ansätze.  
Der State, den wir nicht selbst verwalten wollen, liegt in der `<Menu>`-Komponente. Die Komponenten kommunizieren über [React Context](https://react.dev/learn/passing-data-deeply-with-context), dadurch sind die Implementierungsdetails komplett gekapselt. Wir können uns ganz auf unsere eigene Implementierung konzentrieren.

```jsx
function Example() {
  const download = () => {
    // Download stuff ...
  };

  return (
    <Menu closeOnItemClick>
      <Menu.Button>Menu</Menu.Button>
      <Menu.List>
        <Menu.Item onClick={download}>Download</Menu.Item>
        <Menu.Link to="details">Show details</Menu.Link>
      </Menu.List>
    </Menu>
  );
}
```

Das liest sich deutlich besser. 😍

#### Vorteile

1. Du musst den State nicht selbst verwalten, kannst je nach Implementierung aber trotzdem darauf zugreifen (über Render Props)
2. Die Datenstruktur spielt keine Rolle mehr
3. Die Komponenten sind nicht eng miteinander gekoppelt

Seit React 19 renderst du den Context selbst als Provider: `<MenuContext value={state}>` statt `<MenuContext.Provider value={state}>`. Die einzelnen Teile lesen ihn mit `useContext(MenuContext)`.

Doku: [react.dev: createContext](https://react.dev/reference/react/createContext), [react.dev: useContext](https://react.dev/reference/react/useContext)

**Hilfreiche Links**

- [Passing Data Deeply with Context](https://react.dev/learn/passing-data-deeply-with-context)

## Beispiel: Accordion

Hier sind verschiedene Arten, ein Accordion zu nutzen, das als Compound Component gebaut ist.

**Accordion mit statischen Daten**

Über Render Props kommst du auch an Daten, die das Accordion verwaltet. Das ist optional, du musst die Render Props nicht nutzen. Wie du sie einsetzt, zeigt die unten verlinkte Demo.

```jsx
function App() {
  return (
    <Accordion multiple>
      <Accordion.Item id="1">
        {({ id, isActive }) => (
          <>
            <Accordion.Toggle>
              Section {id} {isActive ? "(active)" : null}
            </Accordion.Toggle>
            <Accordion.Panel>
              <p>Lorem ipsum dolor sit amet consectetur adipisicing elit. Magnam, voluptatum.</p>
            </Accordion.Panel>
          </>
        )}
      </Accordion.Item>
      <Accordion.Item id="2">
        <Accordion.Toggle>Section 2</Accordion.Toggle>
        <Accordion.Panel>
          <p>Lorem ipsum dolor sit amet consectetur adipisicing elit. Magnam, voluptatum.</p>
        </Accordion.Panel>
      </Accordion.Item>
      <Accordion.Item id="3">
        <Accordion.Toggle>Section 3</Accordion.Toggle>
        <Accordion.Panel>
          <p>Lorem ipsum dolor sit amet consectetur adipisicing elit. Magnam, voluptatum.</p>
        </Accordion.Panel>
      </Accordion.Item>
    </Accordion>
  );
}
```

**Accordion mit Daten aus einem Array**

```jsx
function App() {
  // The data can come from anywhere and be used in any structure
  const data = [
    {
      id: "1",
      title: "Section 1",
      description: <p>Lorem ipsum 1</p>
    },
    {
      id: "2",
      title: "Section 2",
      description: <p>Lorem ipsum 2</p>
    },
    {
      id: "3",
      title: "Section 3",
      description: <p>Lorem ipsum 3</p>
    }
  ];

  return (
    <Accordion multiple>
      {data.map((entry) => (
        <Accordion.Item key={entry.id} id={entry.id}>
          <Accordion.Toggle>{entry.title}</Accordion.Toggle>
          <Accordion.Panel>{entry.description}</Accordion.Panel>
        </Accordion.Item>
      ))}
    </Accordion>
  );
}
```

**Demo** 🤯

- [Accordion Compound Component (React)](https://github.com/christiansany/react-compound-components)

**Hilfreiche Links**

- [React Hooks: Compound Components](https://kentcdodds.com/blog/compound-components-with-react-hooks)
- [Passing data with a render prop](https://react.dev/reference/react/cloneElement#passing-data-with-a-render-prop)
- [Advanced React Component Patterns : Render Props](https://medium.com/@pramonowang/advanced-react-component-patterns-render-props-a9d61376a8fb)

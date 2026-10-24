# Compound Components with React

> A component that takes one big configuration object is hard to read and hard to change. Compound components split it into parts you compose in JSX, like `<select>` and `<option>` in HTML, while the parent keeps the state. We build a menu three ways and compare them, then look at an accordion built with the pattern.

**Duration:** about 45 minutes

## Contents

* [What are compound components?](#what-are-compound-components)
* [The problem](#the-problem)
* [The solution](#the-solution)
* [Example: Accordion](#example-accordion)

## What are compound components?

Compound components are a pattern that encapsulates the **state** and the **behaviour** of a group of components, but leaves control over rendering the individual parts to the developer.

The goal of compound components is to provide a more expressive and flexible API for communication between the parent component and its child components.

A good example is `<select>` combined with `<option>` in HTML:

```html
<select>
  <option value="value1">name1</option>
  <option value="value2">name2</option>
  <option value="value3">name3</option>
</select>
```

The two elements communicate with each other in the background. The `<select>` holds the state, and the `<option>`s define what can be selected. This allows expressive markup, and you don't have to pass the options in an attribute.

```html
<!-- With an attribute 😱 -->
<select options="name1:value1;name2:value2;name3:value3"></select>
```

## The problem

Let's say we need to build a menu. The menu has a button that opens and closes it. It contains links and buttons. A click on one of them should trigger the action and close the menu again.

Let's look at different ways to implement it.

### Configuration

```jsx
function Example() {
  const menuConfig = {
    menuButton: {
      text: "Menu",
    },
    menuList: [{
      kind: "item",
      text: "Download",
      onClick: () => { /* ... */ }
    }, {
      kind: "link",
      text: "Show details",
      href: "/details"
    }],
    closeOnItemClick: true,
  };

  return (
    <Menu {...menuConfig} />
  );
}
```

In this example we build a complicated configuration object. It already contains all the information that every component inside `<Menu>` needs for rendering and behaviour. This is a widespread pattern.

The `<Menu>` component manages whether the menu is open or closed itself, and the child components can change the state of `<Menu>`.

#### Pros

1. `<Menu>` manages its own internal state, whether it is open or closed

#### Cons

1. `<Menu>` is directly responsible for rendering all child components, which couples the components very tightly
2. The `props` are passed down through several components. This is called "prop drilling" and is generally considered a code smell
3. If the data comes in a different format, you first have to map it to the structure of the config

```jsx
function Example() {
  // Data that comes from an API or is passed in as props
  const links = [
    { title: "Show details", target: "/details" },
    { title: "Show profile", target: "/profile" }
  ];

  const menuConfig = {
    menuButton: {
      text: "Menu",
    },
    menuList: [{
      kind: "item",
      text: "Download",
      onClick: () => { /* ... */ }
    }].concat(links.map(link => ({ // Data has to be mapped to the structure of the config
      kind: "link",
      text: link.title,
      href: link.target
    }))),
    closeOnItemClick: true,
  };

  return (
    <Menu {...menuConfig} />
  );
}
```

### Managing the state yourself

```jsx
function Example() {
  const [isOpen, setIsOpen] = useState(false);

  const open = () => {
    setIsOpen(true);
  }

  const close = () => {
    setIsOpen(false);
  }

  const download = () => {
    // Download stuff ...
    close();
  }

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
        {links.map(link => (
          <MenuLink key={link.target} to={link.target} onClick={close}>{link.title}</MenuLink>
        ))}
      </MenuList>
    </Menu>
  );
}
```

In this example we don't have to build a complicated configuration object. But because the components are not connected to each other, we have to manage the state of the menu ourselves.

#### Pros

1. The components are no longer tightly coupled, and `<Menu>` no longer has to take care of rendering its child components
2. The data structure no longer matters

#### Cons

1. You have to manage the state yourself. Depending on the component, this gets very complicated

## The solution

The compound components pattern combines both approaches.  
The state we don't want to manage ourselves lives in the `<Menu>` component. The components communicate through [React Context](https://react.dev/reference/react/useContext), so the implementation details are fully encapsulated. We can focus entirely on our own implementation.

```jsx
function Example() {
  const download = () => {
    // Download stuff ...
  }

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

This reads much better.

#### Pros

1. You don't have to manage the state yourself, but depending on the implementation you can still access it (through render props)
2. The data structure no longer matters
3. The components are not tightly coupled

Since React 19, you render the context itself as the provider: `<MenuContext value={state}>` instead of `<MenuContext.Provider value={state}>`. The parts read it with `useContext(MenuContext)`.

Docs: [react.dev: createContext](https://react.dev/reference/react/createContext), [react.dev: useContext](https://react.dev/reference/react/useContext)

**Useful links**

* [Passing Data Deeply with Context](https://react.dev/learn/passing-data-deeply-with-context)

## Example: Accordion

Here are different ways to use an accordion built as a compound component.

**Accordion with static data**

With render props you can also access data that the accordion manages. This is optional, and you don't have to use the render props. To see how to use these optional render props, look at the demo linked below.

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
              <p>
                Lorem ipsum dolor sit amet consectetur adipisicing elit.
                Magnam, voluptatum.
              </p>
            </Accordion.Panel>
          </>
        )}
      </Accordion.Item>
      <Accordion.Item id="2">
        <Accordion.Toggle>Section 2</Accordion.Toggle>
        <Accordion.Panel>
          <p>
            Lorem ipsum dolor sit amet consectetur adipisicing elit. Magnam,
            voluptatum.
          </p>
        </Accordion.Panel>
      </Accordion.Item>
      <Accordion.Item id="3">
        <Accordion.Toggle>Section 3</Accordion.Toggle>
        <Accordion.Panel>
          <p>
            Lorem ipsum dolor sit amet consectetur adipisicing elit. Magnam,
            voluptatum.
          </p>
        </Accordion.Panel>
      </Accordion.Item>
    </Accordion>
  );
}
```

**Accordion with data from an array**

```jsx
function App() {
  // The data can come from anywhere and be used in any structure
  const data = [{
    id: "1",
    title: "Section 1",
    description: (<p>Lorem ipsum 1</p>)
  }, {
    id: "2",
    title: "Section 2",
    description: (<p>Lorem ipsum 2</p>)
  }, {
    id: "3",
    title: "Section 3",
    description: (<p>Lorem ipsum 3</p>)
  }];

  return (
    <Accordion multiple>
      {data.map(entry => (
        <Accordion.Item key={entry.id} id={entry.id}>
          <Accordion.Toggle>
            {entry.title}
          </Accordion.Toggle>
          <Accordion.Panel>
            {entry.description}
          </Accordion.Panel>
        </Accordion.Item>
      ))}
    </Accordion>
  );
}
```

**Demo** 🤯

- [Accordion Compound Component (React)](https://github.com/christiansany/react-compound-components)

**Useful links**

* [React Hooks: Compound Components](https://kentcdodds.com/blog/compound-components-with-react-hooks)
* [Passing data with a render prop](https://react.dev/reference/react/cloneElement#passing-data-with-a-render-prop)
* [Advanced React Component Patterns : Render Props](https://medium.com/@pramonowang/advanced-react-component-patterns-render-props-a9d61376a8fb)

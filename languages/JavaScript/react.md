# React Style Guide

## Use semantic elements in React

In HTML5 there are many elements that have semantic meaning like `<section>` and `<footer>`. There are also common elements that will appear throughout the codebase like `<a>` and `<button>` these should be preferred over `<div>` wherever possible because they carry semantic meaning and it's inherently better for accessibility.

### ℹ️ Why: **Accessibility**

A webpage is often created to be visual but the markup used to create the page is even more important. Using elements that aren't intended to be clickable means you lose a lot of the benefits of semantic HTML. To correct these, you have to ensure the element has aria attributes, a tab index, focus and hover states etc. Using the correct semantic elements also has the benefit of making it easier for accessibility and can be consumed by screen readers.

### ❓ When: **Wherever there exists a more specific element**

A common case for using non-semantic elements is for styling purposes. This is unnecessary, each browser applies a its own style to `<button>` and `<a>` elements. You should override these styles rather than using a more generic element.

### ❌ Don't: **Add click handlers to non-clickable elements**

Avoid adding click handlers to elements that are not clickable by nature

```jsx
<div onClick={handleEvent} />
<img onClick={handleEvent} />
```

If an image should be clickable, wrap it in a clickable element like so:

```jsx
<button onClick={handleEvent}>
  <img />
</button>
```

### ✅ Do: **Use `<a>` or an appropriate link component, for navigation.**

`<a>` should be preferred for navigation, an added benefit of suing the semantically correct elements is that users can open them in a new tab easily. It also conforms to the design patterns of the web.

```jsx
<a href="/car-insurance/">Car Insurance</a>
<Link to="car-insurance">Car Insurance</Link>
```

Use `<button>` elements for actions that don't perform a navigation and/or rely on state.

```jsx
<button onClick={showDialog} />
<button onClick={dismissBanner} />
```

### 🤖 ESLint Rule: **eslint-plugin-jsx-a11y**

Plugin available: [eslint-plugin-jsx-a11y](https://www.npmjs.com/package/eslint-plugin-jsx-a11y) that covers most of these rules.

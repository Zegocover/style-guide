# JavaScript Style Guide

Some of the guidelines will have technical reasons, others will be purely about ensuring consistency. The reasons will be explained where possible. All rules may be broken in certain circumstances, but doing so should be justified by a code comment and/or in the pull request.

This document doesn't cover purely preferential things that are already enforced by tooling (e.g tabs vs spaces).

---

## Migrating to TypeScript

The current website frontend is mostly JavaScript but whenever possible, new files should be written in TypeScript.

### Why TypeScript?

Static type checking is a feature which allows teams to refactor and move quickly with confidence. It helps with onboarding new engineers, improves developer experience and is another layer of trust within the codebase.

### The process of migration

1. **New** files should be created as TypeScript unless there is good reason not to.
2. **Existing** files should only be updated if there is a dedicated refactor to do so or when they are updated as part of other work. If a file is touched and it's feasible to do so, it should be converted to TypeScript.

### Escape hatches

There are ways to avoid typechecking when using TypeScript such as `Any` and `@ts-ignore`. For incremental adoption, it's not a strict requirement to have everything strongly typed but it's encouraged. Loose types aren't trustworthy and can lead to a false sense of trust in the codebase. Use of `Any` should be avoided and `@ts-ignore` should be avoided unless there is no known type and it's not feasible to create one.

### Getting Started with TypeScript

It's quick to become productive in TypeScript, if you're unfamiliar with it then take a look at [TypeScript for JavaScript Programmers](https://www.typescriptlang.org/docs/handbook/typescript-in-5-minutes.html).

### How do I use a JS/JSX component in my TS file?

There are two approaches to take during the transition from JavaScript to TypeScript as the primary development language.

#### Ideal Solution

If you have the time and the inclination, you could convert the component you're importing into a TS file and add types. When doing this as part of related work, it's best to create an additional PR to make it easier for reviewers to digest.
**If you don't have time or it's out of scope, take the practical approach.**

#### Practical Solution

A quick fix is to add a `/* @ts-ignore */` directive above the component usage like so (assumes `PrimaryPrice` is a JS component):

```jsx
<Spacer height={unit(2)} mobileHeight={unit(3)} />
{/* @ts-ignore */}
<PrimaryPrice price={price} />
```

This approach is preferred when it's not feasible to change the components you're importing. When a component fincally makes the transition to TS, any `/* @ts-ignore */` directives should be removed where this new TS component is referenced.

## Formatting and Syntax

### Naming

```
ClassNamesLikeThis
ReactComponentsLikeThis
methodNamesLikeThis
variableNamesLikeThis
parameterNamesLikeThis
propertyNamesLikeThis
SYMBOLIC_CONSTANTS_LIKE_THIS
```

ℹ️ **Why:** Consistency.

### File names

If a module has a default export, eg a React Component, the file name should be the same as the export. If a module has multiple named exports, it should use lowerCamelCase. Use the `.js` file extension unless it's a different language (like TypeScript).

```javascript
// MyComponent.js

import React from 'react';

export default function MyComponent(props) {
  // do stuff
}
```

```javascript
// utils.js

export function doFoo() {}
export function doBar() {}
```

ℹ️ **Why:** Consistency, primarily. But additionally, unexpected casing can cause issues in different environments, eg `button.js` and `Button.js` are the same on MacOS, but different on other systems.

### Indentation and other formatting

Rather than enumerating each formatting preference like indentation, we've opted to use [Prettier](https://prettier.io/) to format code automatically. And [EditorConfig](https://editorconfig.org/) to allow us to automatically configure our editors to adopt the preferred formatting.

Each project should have configuration for both of these tools, and your editor should be configured to use them. CI should be configured to fail on incorrectly formatted code (if there's a legitimate reason to break formatting, you can disable the lint rule on a per-line basis).

ℹ️ **Why:** Consistency. There are very few technical reasons to favour one formatting convention over another, but there's value in picking one and sticking with it.

### Imports

We have a loose convention for ordering of imports, each group is usually separated by a blank line:

1. React itself, for any module that uses JSX.
2. System modules (only applicable for Node code), it's rare to see these in a module containing React code, so in reality these will come first.
3. Architectural modules. These are modules that get used across a huge proportion of the codebase, e.g. Relay, emotion and lodash.
4. Incidental third-party modules. These are modules which get used to fulfill specific module-level use cases,
5. Shared first-party modules, e.g. a React component that's reused in numerous places (like a `<Button>`), imported from the configured `@zego` namespace
6. Local first-party modules. Usually a sibling of the module, or within a sibling directory, tied to a specific use-case rather than of general use.
7. Non-JS imports, eg JSON and images.

Imports within each group should be sorted alphabetically.

ℹ️ **Why:** Consistency, and ease of understanding.

### Do _not_ sort imports automatically

Whilst it's fine to have a command you run manually to order imports (according to conventions) within a single module, it **shouldn't** be configured to run automatically (e.g. on save).

ℹ️ **Why:** Importing a module causes it to be executed, it's possible for this to include side effects. In some circumstances it can be necessary to carefully sort imports so that the code functions as expected. Automatically sorting imports can cause code to break.

### Linting

Each project should be configured for [ESLint](https://eslint.org/), and your editor should be configured to show lint errors/warnings. We usually use an external configuration as a foundation for our own (eg [eslint-config-react-app](https://www.npmjs.com/package/eslint-config-react-app)), so it's possible that some rules are too strict or lenient by default. The aim should be to get the total number of errors and warnings to 0.

An error usually means that something is likely broken and needs fixing. Our builds are configured to fail when there are errors. A warning means something is undesirable, but likely won't break anything. An unused import probably won't break the website, but could increase the size of our code bundles , and therefore should be avoided.

Some linting violations are able to be fixed automatically, but these should be done on a case-by-case basis rather than across the entire codebase.

If a linting rule seems incorrect or excessively draconian, there should be a discussion about removing it.

ℹ️ **Why:** Linting protects us from bugs and other forms of codebase deterioration. A pull request containing linting violations creates extra work for the reviewers.

---

## Best practices, patterns and anti-patterns

We don't have linting and formatting rules for everything, so let's
dive into some specifics.

### Ternaries

✅ **Do:** Use ternaries for simple boolean expressions.

_Evaluates to 'green' if selected is truthy, 'orange' otherwise_

```javascript
selected ? 'green' : 'orange';
```

It's possible to build up more elaborate ternaries by nesting them, try not to go beyond one level of nesting because it can become quite hard to understand them.

✅ **Do:** Use at most one level of ternary nesting,

This example is a borderline one, it's already getting hard to understand.

_Evaluates to 'grey' if disabled is truthy, falling back to 'green' if selected is truthy, 'orange' otherwise._

```javascript
disabled ? 'grey' : selected ? 'green' : 'orange';
```

❌ **Don't:** Use ternary nesting if it compromises readability

_This is too much._

```javascript
disabled ? 'grey' : selected ? 'green' : focused ? 'blue' : 'orange';
```

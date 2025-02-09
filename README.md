# rollup-plugin-jsx-if-for
This is a plugin for [the Rollup bundler](https://rollupjs.org) which
rewrites [JSX content](https://react.dev/learn/writing-markup-with-jsx)
(whether or not React is used) so certain "pseudo-component" tags are
converted to corresponding Javascript expressions:

- `<$if test={expr}>...</$if>` becomes `{(expr) ? <>...</> : null}`
- `<$for var="id" of={expr}>...</$for>` becomes `{(expr).map((id) => <> ... </>)}`
- `<$let var="id" value={expr}>...</$let>` becomes `{((id) => <> ... </>)(expr)}`

Note that `var` in `<$for>` and `<$let>` may be a variable name or a
destructuring pattern, and `of` and `value` may be any Javascript expression:
```jsx
<$for var="{x, y}" of {[{x: 1, y: 2}, {x: 3, y: 4}]}>
  <div>x is {x}, y is {y}</div>
</$for>
```

## Why?

In most contexts you can and should write the `{...}` equivalent directly.

However, [MDX](https://mdxjs.com/) (Markdown with JSX support)
only allows Markdown content inside component tags, not inside Javascript
curly braces. (See
[these](https://github.com/orgs/mdx-js/discussions/2581)
[discussions](https://github.com/orgs/mdx-js/discussions/2276).)

So, while this plugin isn't technically MDX-specific, it exists mostly to
deal with this MDX quirk and let you write conditions, loops, and local
variable bindings around Markdown content. ("Traditional" template
languages often use tag-based conditionals and loops in this way.)

## Usage

Add this module:
```sh
npm i rollup-plugin-jsx-if-for
```

Configure Rollup to use it, in `rollup.config.js` or equivalent:
```js
import rollupJsxIfFor from "rollup-plugin-jsx-if-for";
...
export default {
  ...
  plugins: [
    ...
    rollupJsxIfFor({ ... }),
  ],
};
```

This plugin's constructor takes
[conventional](https://rollupjs.org/plugin-development/#transformers)
`include` and `exclude` options to subselect from the files
Rollup is processing.

```js
rollupJsxIfFor({
  include = ["**/*.mdx", "**/*.jsx"],
  exclude = [],
})
```

> [!NOTE]
> List this plugin AFTER plugins which convert other formats into JS/JSX (eg.
> [`rollup-plugin-postcss`](https://github.com/egoist/rollup-plugin-postcss#readme)),
> or in fact [`@mdx-js/rollup`](https://mdxjs.com/packages/rollup/).
> Otherwise this plugin will fail trying to parse CSS or raw MDX or something.

## Using this plugin with MDX

If you're using this plugin, you're probably also using
[`@mdx-js/rollup`](https://mdxjs.com/packages/rollup/).

Configure both
[MDX](https://mdxjs.com/packages/mdx/#processoroptions)
and
[Rollup](https://rollupjs.org/configuration-options/#jsx)
so JSX-to-JS conversion happens in the Rollup core, not the MDX plugin:

```js
import rollupJsxIfFor from "rollup-plugin-jsx-if-for";
import rollupMdx from "@mdx-js/rollup";
...
export default {
  ...
  jsx: { mode: "automatic", jsxImportSource: ... },
  ...
  plugins: [
    ...
    rollupMdx({ jsx: true }),  // output JSX tags in MDX output
    rollupJsxIfFor({ ... }),
    ...
  ],
};
```

## Minimal example

Use this `rollup.config.mjs`

```js
import fg from "fast-glob";
import rollupAutomountDom from "rollup-plugin-automount-dom";
import rollupHtml from "@rollup/plugin-html";
import rollupJsxIfFor from "rollup-plugin-jsx-if-for";
import rollupMdx from "@mdx-js/rollup";
import { nodeResolve as rollupNodeResolve } from "@rollup/plugin-node-resolve";

export default fg.sync("*.mdx").map((input) => ({
  input,
  jsx: { mode: "automatic", jsxImportSource: "jsx-dom" },
  output: { directory: "dist", format: "iife" },
  plugins: [
    rollupAutomountDom(),
    rollupHtml({ fileName: input.replace(/mdx$/, "html"), title: "" }),
    rollupJsxIfFor(),
    rollupMdx({ jsx: true }),
    rollupNodeResolve(),
  ],
}));
```

And this `beer.mdx`

```mdx
<$let var="countdown" value={[...Array(100).keys()].reverse()}>
  <$for var="n" of={countdown}>
    <$if test={n > 0}>
      ## {n} bottles of beer on the wall, {n} bottles of beer
      Take one down and pass it around,
      <$if test={n > 1}>{n - 1}</$if> <$if test={n <= 1}>no more</$if>
      bottles of beer on the wall.
    </$if>
    <$if test={n == 0}>
      ## No more bottles of beer on the wall, no more bottles of beer
      Go to the store and buy some more, 99 bottles of beer on the wall
    </$if>
  </$for>
</$let>
```

And then run

```sh
npm i fast-glob jsx-dom @mdx-js/rollup rollup rollup-plugin-automount-dom @rollup/plugin-html rollup-plugin-jsx-if-for @rollup/plugin-node-resolve
npx rollup -c
```

And finally load `dist/beer.html` in your browser and you should see something
like this

![image](https://github.com/user-attachments/assets/18db5fb4-bfc6-4eed-883e-530b8c6a65c0)

Tada! 🍺

## Tips and Pitfalls

### ⚠️ Don't use MDX `export` inside tag scopes (use `<$let>` instead)

In MDX content, `<$if>`, `<$for>`, and `<$let>` tags will wrap Markdown/JSX,
BUT ALL `export` directives are executed globally first. This will NOT work:

```mdx
<$for var="i" of={[1, 2, 3]}>
  export const j = i * 2;  // WILL FAIL, is evaluated BEFORE and OUTSIDE the loop
  ## {i} times 2 is {j}   {/* WILL NOT WORK */}
</$for>
```

Instead, use `<$let>` for local bindings, like this:
```mdx
<$for var="i" of={[1, 2, 3]}>
  <$let var="j" value={i * 2}>
    ## {i} times 2 is {j}
  </$let>
</$for>
```

### ℹ️ Ways to avoid nested `<$let>` towers

If you find yourself with towers of annoyingly nested dependent `<$let>` tags:
```mdx
<$let var="x" value={3.14159}>
  <$let var="y" value={x * x}>
    <$let var="z" value={y / (x + 1)}>
      ## x={x} y={y} z={z}
    </$let>
  </$let>
</$let>
```

Consider instead building an object in an immediately invoked function:
```mdx
<$let var="{x, y, z}" value={(() =>
  const x = 3.14159;
  const y = x * x;
  const z = y / (x + 1);
  return {x, y, z};
)()}>
  ## x={x} y={y} z={z}
</$let>
```

You could also use a named function with MDX `export` (if the function can run in global scope):

```mdx
export function getXYZ() {
  const x = 3.14159;
  const y = x * x;
  const z = y / (x + 1);
  return {x, y, z};
}

...
<$let var="{x, y, z}" value={getXYZ()}>
  ## x={x} y={y} z={z}
</$let>
```

You could even `import` the function from another module entirely, if that makes sense for you.

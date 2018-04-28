# Specification

[MDX](https://github.com/mdx-js/mdx) language and abstract syntax tree definitions.

- [Why?](#why)
- [How does it work?](#how-does-it-work)
- [MDX](#mdx)
- [MDXAST](#mdxast)
- [MDXHAST](#mdxhast)
- [Plugins](#plugins)
- [Related](#related)
- [Authors](#authors)

## Why?

In order to ensure a vibrant ecosystem and community, tooling needs to exist for formatting, linting, and plugins.
This tooling requires a foundational specification and abstract syntax tree so that parsing is properly handled before transforming to JSX/Hyperscript/React/etc and potentially leveraging existing plugin ecosystems.

[mdx-js/mdx](https://github.com/mdx-js/mdx) uses Remark to parse Markdown into an MDAST which is transpiled to MDXAST.
This allows for the rich [unified](https://github.com/unifiedjs) plugin ecosystem to be utilized while also ensuring a more robust parsing implementation by sharing the Remark parser library.

## How does it work?

The MDX transpilation flow consists of six steps, ultimately resulting in JSX that can be used in React/Preact/etc.

1. _Parse_: Text => MDAST
1. _Transpile_: MDAST => MDXAST
1. _Transform_: MDX/Remark plugins applied to AST
1. _Transpile_: MDXAST => MDXHAST
1. _Transform_: Hyperscript plugins applied to AST
1. _Transpile_: MDXHAST => JSX

## MDX

MDX is superset of the [CommonMark](http://commonmark.org) specification that adds embedded JSX and `import`/`export` syntax.

### Imports

ES2015 [`import` syntax](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/import) is supported.
This can be used to transclude other MDX files or to import React components to render.

#### MDX transclusion

Shared content can be transcluded by using `import` syntax and then rendering the component.
Imported MDX is transpiled to a React/JSX component at compile time.

```js
import License from './shared/license.md'

## Hello, world!

<License />
```

#### React component rendering

```js
import Video from './video'

## Hello, world!

<Video />
```

### Exports

ES2015 [`import` syntax](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/export) is supported.
This can be used to export metadata like layout or authors.
It's a mechanism for an imported MDX file to communicate with its parent.

```js
import { fred, sue } from '../data/authors'
import Layout from '../components/with-blog-layout'

export const meta = {
  authors: [fred, sue],
  layout: Layout
}
```

### JSX

In MDX, all embedded markup is interpreted as JSX.
Components can be imported for rendering.

#### Block level

JSX can be written at the block level.
This means that it is rendered as a sibling to other root level elements like paragraphs and headings.
It must be surrounded by empty newlines.

```jsx
import { Logo } from './ui'

# Hello, world!

<Logo />

And here's a paragraph
```

### Element to component mapping

It's often be desirable to map React components to their HTML element equivalents, adding more flexibility to many usages of React that might not want a plain HTML element as the output.
This is useful for component-centric projects.

```jsx
import React from 'react'
import * as ui from './ui'

import Doc from './readme.md'

export default () =>
  <Doc
    components={{
      h1: ui.Heading,
      p: ui.Text,
      code: ui.Code
    }}
  />
```

## MDXAST

The majority of the MDXAST specification is defined by [MDAST](https://github.com/syntax-tree/mdast).
MDXAST is a superset of MDAST, with three additional node types:

- `jsx` (in place of `html`)
- `import`
- `export`

It's also important to note that an MDX document that contains no JSX or imports is a valid MDAST.

### Differences to MDAST

The `import` type is used to provide the necessary block elements to the Remark HTML block parser and for the execution context/implementation.
For example, a [webpack loader](https://github.com/mdx-js/mdx/tree/master/packages/loader) might want to transform an MDX import by appending those imports.

`export` is used to emit data from MDX, similarly to traditional markdown frontmatter.

The `jsx` node would most likely be passed to Babel to create functions.

This will also differ a bit in parsing because the remark parser is built to handle particular HTML element types, whereas JSX support will require the ability to parse _any_ tag, and those that self close.

The `jsx`, `import`, and `export` node types are defined below.

### AST

#### JSX

`JSX` ([`ElementNode`](#elementnode)) which contains embedded JSX as a string and `children` ([`ElementNode`](#elementnode)).

```idl
interface JSX <: Element {
  type: "jsx";
  value: "string";
  children: [ElementNode]
}
```

For example, the following MDX:

```jsx
<Heading hi='there'>
  Hello, world!
</Heading>
```

Yields:

```json
{
  "type": "jsx",
  "value": "<Heading hi='there'>\n  Hello, world!\n</Heading>"
}
```

#### Import

`import` ([`Textnode`](#textnode)) contains the raw import as a string.

```idl
interface JSX <: Text {
  type: "import";
}
```

For example, the following MDX:

```md
import Video from '../components/Video'
```

Yields:

```json
{
  "type": "import",
  "value": "import Video from '../components/Video'"
}
```

#### Export

`export` ([`Textnode`](#textnode)) contains the raw export as a string.

```idl
interface JSX <: Text {
  type: "export";
}
```

For example, the following MDX:

```md
export { foo: 'bar' }
```

Yields:

```json
{
  "type": "export",
  "value": "export { foo: 'bar' }"
}
```

## MDXHAST

The majority of the MDXHAST specification is defined by [HAST](https://github.com/syntax-tree/hast).
MDXHAST is a superset of HAST, with four additional node types:

- `jsx`
- `import`
- `export`
- `inlineCode`

It's also important to note that an MDX document that contains no JSX or imports results in a valid HAST.

## Plugins

The `@mdx-js/mdx` implementation is pluggable at multiple stages in the transformation.
This allows not only for retext/remark/rehype transformations, but also access to the powerful utility libraries these ecosystems offer.

Remark/MDX processing is async, so any plugin that returns a promise will be `await`ed.

### MDAST plugins

When the MDX library receives text, it uses `remark-parse` to parse the raw content into an MDAST.
MDX then uses a few formatting plugins to ensure the MDAST is cleaned up.
At this stage, users have the ability to pass any (optional) plugins to manipulate the MDAST.

```js
const jsx = mdx(myMDX, { mdPlugins: [myPlugin] })
```

Let's consider the following default `remark-images` plugin that MDX uses.
This plugin automatically turns a paragraph that consists of only an image link into an image node.

```js
const isUrl = require('is-url')
const visit = require('unist-util-visit')

const isImgUrl = str => /\.(svg|png|jpg|jpeg)/.test(str)

module.exports = () => (tree, file) =>
  visit(tree, 'text', node => {
    const text = node.value ? node.value.trim() : ''

    if (!isUrl(text) || !isImgUrl(text)) {
      return
    }

    node.type = 'image'
    node.url = text

    delete node.value
  })
```

Not bad. The `unist-util-visit` utility library makes it terse to select nodes we care about, we check it for a few conditions, and manipulate the node if it's what we're looking for.

### HAST plugins

HAST plugins operate similarly to MDAST plugins, except they have a different AST spec that defines it.

```js
const jsx = mdx(myMDX, { hastPlugins: [myPlugin] })
```

Let's consider another example plugin which asynchronously requests an image's size and sets it as attributes on the node.

```js
await mdx(myMDX, {
  hastPlugins: [
    () => tree => {
      const imgPx = selectAll('img', tree).map(async node => {
      const size = await requestImageSize(node.properties.src)
        node.properties.width = size.width
        node.properties.height = size.height
      })

      return Promise.all(imgPx).then(() => tree)
    }
  ]
})
```

Note that this might want to also take an optional argument for the max width of its container for a truly solid layout, but this is a proof of concept.

## Related

This specification documents the [original `.mdx` proposal](https://spectrum.chat/thread/1021be59-2738-4511-aceb-c66921050b9a) by Guillermo Rauch ([@rauchg](https://twitter.com/rauchg)).

The following projects, languages, and articles helped to shape MDX either in implementation or inspiration.

### Syntax

These projects define the syntax which MDX blends together (MD and JSX).

- [Markdown](https://daringfireball.net/projects/markdown/syntax)
- [JSX](https://reactjs.org/docs/introducing-jsx.html)
- [React](https://reactjs.org/)

### Parsing and implementation

- [Remark](http://remark.js.org)
- [Unified](https://github.com/unifiedjs/unified)
- [Webpack](https://webpack.js.org)

### Libraries

- [MDXC](https://github.com/jamesknelson/mdxc)
- [remark-jsx](https://github.com/fazouane-marouane/remark-jsx)
- [remark-react](https://github.com/mapbox/remark-react)

### Other

- [IA Markdown Content Blocks](https://github.com/iainc/Markdown-Content-Blocks)

### Is your work missing?

If you have related work or prior art we've failed to reference, please open a PR!

## Authors

- John Otander ([@4lpine](https://twitter.com/4lpine)) – [Compositor](https://compositor.io) + [Clearbit](https://clearbit.com)
- Tim Neutkens ([@timneutkens](https://github.com/timneutkens)) – [ZEIT](https://zeit.co)
- Guillermo Rauch ([@rauchg](https://twitter.com/rauchg)) – [ZEIT](https://zeit.co)
- Brent Jackson ([@jxnblk](https://twitter.com/jxnblk)) – [Compositor](https://compositor.io)

# Specification

[MDX](https://github.com/mdx-js/mdx) language and abstract syntax tree definitions.

- [Why?](#why)
- [How does it work?](#how-does-it-work)
- [MDX](#mdx)
- [MDXAST](#mdxast)
- [MDXHAST](#mdxhast)
- [Related](#related)

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

export {
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

### Images

Embedding images is easier to remember, you can simply link a url.

```md
Below will render an image:

https://c8r-x0.s3.amazonaws.com/lab-components-macbook.jpg
```

The following file types are supported:

- `png`
- `svg`
- `jpg`

### Element to component mapping

It's often be desirable to map React components to their HTML element equivalents, adding more flexibility to many usages of React that might not want a plain HTML element as the output.
This is useful for component-centric projects using CSS-in-JS and other projects.

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

The `import` type is used to provide the necessary block elements to the remark html block parser and for the execution context/implementation.
For example, a [webpack loader](https://github.com/mdx-js/mdx/tree/master/packages/loader) might want to transform an MDX import by appending those imports.

`export` is used to emit data from MDX, similarly to traditional markdown frontmatter.

The `jsx` node would most likely be passed to Babel to create functions.

This will also differ a bit in parsing because the remark parser is built to handle particular html element types, whereas JSX support will require the ability to parse _any_ tag, and those that self close.

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

For example, the following mdx:

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

## MDXHAST

The majority of the MDXHAST specification is defined by [HAST](https://github.com/syntax-tree/hast).
MDXHAST is a superset of HAST, with four additional node types:

- `jsx`
- `import`
- `export`
- `inlineCode`

It's also important to note that an MDX document that contains no JSX or imports results in a valid HAST.

## Related

This specification documents the [original `.mdx` proposal](https://spectrum.chat/thread/1021be59-2738-4511-aceb-c66921050b9a) by Guillermo Rauch ([@rauchg](https://twitter.com/4lpine)).

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

- John Otander ([@4lpine](https://twitter.com/4lpine)) - [Compositor](https://compositor.io) + [Clearbit](https://clearbit.com)
- Tim Neutkens ([@timneutkens](https://github.com/timneutkens)) – [ZEIT](https://zeit.co)
- Guillermo Rauch ([@rauchg](https://twitter.com/rauchg)) – [ZEIT](https://zeit.co)
- Brent Jackson ([@jxnblk](https://twitter.com/jxnblk)) - [Compositor](https://compositor.io)

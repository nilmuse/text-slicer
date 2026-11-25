# text-slicer

Split plain text into word and grapheme `<span>` elements for animation and styling.

[![npm](https://img.shields.io/npm/v/text-slicer.svg?colorB=brightgreen)](https://www.npmjs.com/package/text-slicer)
[![NPM Downloads](https://img.shields.io/npm/dm/text-slicer.svg?style=flat)](https://www.npmjs.com/package/text-slicer)

[Demo](https://codepen.io/ux-ui/full/vYMoGoG)

## Installation

```bash
npm install text-slicer
```

## Quick start

In a browser app with a bundler, add this element to the page:

```html
<p class="text-slicer">Hello world</p>
```

Run this TypeScript after the element exists, for example from your app's module entry:

```ts
import { TextSlicer } from 'text-slicer';

const textSlicer = new TextSlicer({
  container: '.text-slicer',
  splitMode: 'both',
  cssVariables: true,
});

textSlicer.init();
```

This creates `.ts-word` wrappers containing `.ts-char` spans. The `cssVariables` option adds indexes for CSS effects; see [Styling](#styling) for an animation example. TextSlicer does not include a stylesheet.

Before removing or replacing the element, call `textSlicer.destroy()` to restore its plain text and clean up.

## API

```ts
import { TextSlicer, CLASSNAMES } from 'text-slicer';
```

- `TextSlicer` — the main class.
- `CLASSNAMES` — class map for `word`, `char`, and `whitespace`.
- Types: `SplitMode`, `TextSlicerOptions`, `TextSlicerMetrics`, `TextSlicerCallbacks`.

Constructor: `new TextSlicer(options?, callbacks?)`.

### Options

| Option | Type | Default | Description |
|:--|:--|:--:|:--|
| `container` | `HTMLElement \| string` | `undefined` | Target element or selector. One instance manages one element, resolved when it is constructed. Without a matching element, rendering methods do nothing. |
| `splitMode` | `'words' \| 'chars' \| 'both'` | `'both'` | Split into words, graphemes, or both. |
| `cssVariables` | `boolean` | `false` | Sets `--word-index` / `--char-index` on generated spans and `--word-total` / `--char-total` on the container. |
| `dataAttributes` | `boolean` | `false` | Adds `data-word` and `data-char` attributes. |
| `keepWhitespaceNodes` | `boolean` | `true` | When `false`, whitespace stays visible as text nodes instead of `.ts-whitespace` spans in character-based modes. |
| `containerHeightVar` | `boolean` | `false` | Sets `--container-height` from `ResizeObserver` measurements when available. |
| `locale` | `string \| string[]` | `undefined` | Locale passed to `Intl.Segmenter`; omitted means the runtime default locale. |

### Callbacks

| Callback | Arguments | Description |
|:--|:--|:--|
| `onAfterRender` | `TextSlicerMetrics` | Runs after render with `wordTotal`, `charTotal`, and `renderedAt`. |

`charTotal` counts non-whitespace grapheme clusters, not UTF-16 code units. In `chars` and `both` modes it matches rendered `.ts-char` spans. In `words` mode, metrics still describe the original plain text even though character spans are not rendered.

Pass callbacks as the constructor's second argument. To add render metrics and data attributes, replace the initialization from Quick start with:

```ts
const textSlicer = new TextSlicer(
  { container: '.text-slicer', dataAttributes: true },
  { onAfterRender: (metrics) => console.log(metrics) },
);

textSlicer.init();
```

### Methods

| Method | Description |
|:--|:--|
| `init()` | Render the current text and mark the instance mounted. |
| `reinit(newText?, options?)` | Render again, optionally with new plain text and updated options. The container cannot be changed. |
| `clear()` | Remove rendered children and restore library-owned accessibility attributes. Use `destroy()` for full cleanup. |
| `split()` | Render the current text again. |
| `destroy()` | Disconnect observers, restore library-owned styles and attributes, remove generated spans, and put the original plain text back. |
| `updateOptions(options)` | Update options and re-render if mounted. The container cannot be changed; `undefined` option values keep the current setting. |
| `lockHeight()` | Lock the container height to its measured value. |
| `unlockHeight()` | Restore the previous inline `height`. |
| `metrics` | Read metrics computed from the stored plain text. `renderedAt` is the timestamp when the metrics are read. |

## Splitting behavior

- `words`: word-like segments become `.ts-word`; punctuation stays visible but is not counted as a word.
- `chars`: non-whitespace grapheme clusters become `.ts-char`.
- `both`: word-like segments become `.ts-word`, and every non-whitespace grapheme, including punctuation, becomes `.ts-char`.

Word-like segments come from `Intl.Segmenter` with `granularity: 'word'`. Whitespace stays visible as `.ts-whitespace` spans, or as plain text nodes when `keepWhitespaceNodes: false` is used in character-based modes.

## Accessibility

Generated visual spans are marked `aria-hidden="true"`.

If the container has no `aria-label` or `aria-labelledby`, TextSlicer adds a managed `aria-label` with the original plain text and restores it on cleanup.

If you provide your own `aria-label` or `aria-labelledby`, TextSlicer leaves it alone. Keep your custom accessible label in sync when calling `reinit()` with new text.

## Multiple elements

For a page with several matching elements, replace the single-instance setup from Quick start with this code. It uses the same import and runs after the elements exist:

```ts
const textSlicers = Array.from(
  document.querySelectorAll<HTMLElement>('.text-slicer'),
  (container) => {
    const slicer = new TextSlicer({ container });
    slicer.init();
    return slicer;
  },
);

function destroyTextSlicers() {
  textSlicers.forEach((slicer) => slicer.destroy());
}
```

Call `destroyTextSlicers()` before replacing the section or cleaning up a page during navigation.

## Framework integration

Call `init()` only after the element exists in the browser. Call `destroy()` before unmounting or replacing the element.

### React and Next.js

Use a client component in Next.js. A complete component:

```tsx
'use client';

import { useEffect, useRef } from 'react';
import { TextSlicer } from 'text-slicer';

export function AnimatedHeading() {
  const ref = useRef<HTMLHeadingElement>(null);

  useEffect(() => {
    if (!ref.current) return;

    const slicer = new TextSlicer({
      container: ref.current,
      splitMode: 'chars',
      cssVariables: true,
    });

    slicer.init();
    return () => slicer.destroy();
  }, []);

  return <h2 ref={ref}>Hello world</h2>;
}
```

### Vue

```vue
<script setup lang="ts">
import { onBeforeUnmount, onMounted, ref } from 'vue';
import { TextSlicer } from 'text-slicer';

const el = ref<HTMLHeadingElement | null>(null);
let slicer: TextSlicer | undefined;

onMounted(() => {
  if (!el.value) return;
  slicer = new TextSlicer({ container: el.value });
  slicer.init();
});

onBeforeUnmount(() => slicer?.destroy());
</script>

<template>
  <h2 ref="el">Hello world</h2>
</template>
```

### Svelte

```svelte
<script lang="ts">
import { onMount } from 'svelte';
import { TextSlicer } from 'text-slicer';

let node: HTMLHeadingElement;

onMount(() => {
  const slicer = new TextSlicer({ container: node });
  slicer.init();

  return () => slicer.destroy();
});
</script>

<h2 bind:this={node}>Hello world</h2>
```

## Styling

Exported `CLASSNAMES`:

- `ts-word` — word wrapper.
- `ts-char` — character span.
- `ts-whitespace` — whitespace span.

When `cssVariables: true`, `--word-total` and `--char-total` are set on the container. `--word-index` is set on `.ts-word` spans, and `--char-index` is set on non-whitespace `.ts-char` spans.

`--container-height` is based on `ResizeObserver` measurements when `containerHeightVar: true` and the platform API is available.

For the Quick start setup (`cssVariables: true` and `splitMode: 'both'`), add this CSS for a staggered character reveal:

```css
.ts-char {
  display: inline-block;
  opacity: 0;
  transform: translateY(0.5em);
  animation: char-in 0.4s ease forwards;
  animation-delay: calc(var(--char-index) * 40ms);
}

@keyframes char-in {
  to {
    opacity: 1;
    transform: translateY(0);
  }
}
```

## Limitations and browser support

- Plain text only: nested markup is not preserved. `destroy()` restores `textContent`, not the original child nodes. After `reinit(newText)`, cleanup restores that new text.
- Best for short headings, labels, and animation copy. Long text can create a lot of DOM nodes.
- `split()` and a mounted `updateOptions()` rebuild the generated content.
- Construct the instance after its element exists. In SSR frameworks, initialize it after mount in the browser.

`Intl.Segmenter` is required for correct word and grapheme splitting. Without it, TextSlicer uses a simplified fallback; emoji and complex grapheme clusters may be incorrect.

`containerHeightVar` is a no-op when `ResizeObserver` is unavailable.

## License

MIT

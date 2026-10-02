---
description: "Never put HTML tags, custom elements, or angle-bracket markup in any comment"
---

# No HTML / Markup in Comments

Never put HTML tags, custom-element names in angle brackets, attributes, or markup snippets inside **any** comment — including Liquid `{% comment %}`, HTML `<!-- -->`, JS/TS `//` and `/* */`, and CSS `/* */`.

This applies even when the file is `.liquid` or `.html` and the comment is inside a `<script>` or `<style>` block.

## Why

Angle brackets in comments look like live markup, confuse parsers and reviewers, and are a common accidental-uncomment hazard.

## Forbidden in comments

- HTML / custom elements: `<div>`, `<variant-selection>`, `</span>`
- Attribute snippets: `class="hero"`, `data-price-ui`
- Markup fragments or pasted template HTML
- Escaped or partial tags meant to “name” an element: `&lt;div&gt;`, `<div`

## Allowed

Plain language only. Name elements without angle brackets.

```javascript
// BAD
// theme fires variant-change on <variant-selection>; does not bubble

// GOOD
// theme fires variant-change on the variant-selection element; does not bubble
```

```liquid
{% comment %}
BAD — HTML in comment
<div class="gwp-banner">Free gift unlocked</div>
{% endcomment %}

{% comment %}
GOOD — plain language
Shows the free-gift banner when the cart threshold is met.
{% endcomment %}
```

```html
<!-- BAD: <section class="hero">...</section> -->

<!-- GOOD: Hero section wrapper; keep markup in the template body -->
```

## Self-check before finishing an edit

If a comment contains `<` followed by a letter or `/`, rewrite it in plain language. Do not leave the violation and move on.

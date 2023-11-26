---
title: Firefox (Quantum) with dark GTK themes
tags:
  - obsolete
---

## The Problem

Having dark GTK themes active results in Firefox (Quantum) rendering forms and
other stuff unreadable.

## The Solution

Replace the css for all relevant elements!

Create a file `~/.mozilla/firefox/blergbaz.default/chrome/userContent.css` with the following content:

(You have to replace `blergbaz.default` with your profile name, obviously.

```css
button,
input:not([type="checkbox"]):not([type="radio"]),
select,
textarea {
  -moz-appearance: none !important;
  background-color: white;
  color: black;
}
```

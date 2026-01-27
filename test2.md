```ts
boxShadow: "0px 20px 50px rgba(8,112,184,0.7)"
```

This follows the CSS `box-shadow` syntax.

---

## Order of values

```css
box-shadow: offset-x offset-y blur-radius color;
```

---

## Term by term

### `0px` → **offset-x**

* Horizontal shift
* `0px` = shadow is centered (no left/right move)

---

### `20px` → **offset-y**

* Vertical shift
* Positive value = shadow goes **down**
* Makes the element look lifted

---

### `50px` → **blur-radius**

* How soft the shadow is
* Larger value = more spread + softer

---

### `rgba(8,112,184,0.7)` → **color**

Breakdown:

* `8` → red
* `112` → green
* `184` → blue
* `0.7` → opacity (70%)

This gives a **strong cyan-blue glow**

---

## What this shadow visually does

* No side shadow
* Heavy downward depth
* Very soft blur
* Colored glow instead of black

Used for:

* Hover elevation
* Neon / glass UI
* Focused attention

---

## If written fully

```css
box-shadow: 0 20px 50px rgba(8,112,184,0.7);
```

---

### Key difference from Tailwind `shadow-lg`

* `shadow-lg` → fixed, neutral, black
* This → animated, colored, dramatic

Next question when ready.

---
title: Specification
draft: true
---

Korin is a fine-grained reactive TUI framework for Rust. It combines reactive primitives (forked from Leptos) with a DOM-like tree structure, TSS (Terminal Style Sheets) for styling, and Taffy for layout.

## Design Goals

- **Fine-grained reactivity**: Leptos-style signals, effects, memos
- **Familiar mental model**: HTML-like markup, CSS-like styling
- **TUI-native**: Styling and units designed for terminal grids, not pixels
- **ratatui integration**: Renders to ratatui for broad terminal support

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                        User Code                            │
│              view! { <div>...</div> }                       │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    korin_macro                              │
│              view!, #[component]                            │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    korin_view                               │
│         View trait, build/rebuild, cursors, markers         │
│              Show, Each/For, event handlers                 │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                   korin_reactive                            │
│            Signals, Effects, Memos, Runtime                 │
│              (forked from leptos_reactive)                  │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                     korin_dom                               │
│         Document, Node, tree ops, event dispatch            │
│              TSS integration, Taffy layout                  │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                     korin_tss                               │
│       Terminal Style Sheets: cssparser + selectors          │
│         TUI-native properties, cell-based units             │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    korin_render                             │
│         Document → ratatui, event loop, crossterm           │
└─────────────────────────────────────────────────────────────┘
```

---

## Crate: `korin_tss`

Terminal Style Sheets - CSS syntax with TUI semantics.

### Philosophy

TSS uses familiar CSS syntax (parsed via `cssparser`) but with:

- **Cell-based units**: All lengths are in terminal cells. Unitless numbers are the default.
- **Curated properties**: Only properties meaningful in TUI contexts.
- **Terminal colors**: Named colors, ANSI codes, and RGB (where supported).

### Parsing & Grammar

TSS uses `cssparser` for tokenization. Each property defines its own value grammar.

#### Grammar Notation

```
<length>      = <number> | <number>'c' | <percentage> | calc(<calc-expr>)
<percentage>  = <number>'%'
<number>      = [+-]?[0-9]+('.'[0-9]+)?
<integer>     = [+-]?[0-9]+
<color>       = <named-color> | <ansi-color> | <rgb> | <hex>
<named-color> = black | red | green | yellow | blue | magenta | cyan | white
              | bright-black | bright-red | bright-green | bright-yellow
              | bright-blue | bright-magenta | bright-cyan | bright-white
              | reset
<ansi-color>  = ansi(<integer>)  /* 0-255 */
<rgb>         = rgb(<integer>, <integer>, <integer>)
<hex>         = '#'[0-9a-fA-F]{6} | '#'[0-9a-fA-F]{3}
<calc-expr>   = <calc-term> (('+' | '-') <calc-term>)*
<calc-term>   = <calc-factor> (('*' | '/') <calc-factor>)*
<calc-factor> = <number> | <percentage> | '(' <calc-expr> ')'
```

#### Property Value Grammars

**Display:**

```
display = block | flex | grid | inline | none
```

**Dimensions:**

```
width      = <length> | auto
height     = <length> | auto
min-width  = <length>
max-width  = <length> | none
min-height = <length>
max-height = <length> | none
```

**Box Model:**

```
margin         = <length>{1,4}
margin-top     = <length>
margin-right   = <length>
margin-bottom  = <length>
margin-left    = <length>

padding        = <length>{1,4}
padding-top    = <length>
padding-right  = <length>
padding-bottom = <length>
padding-left   = <length>
```

**Border:**

```
border        = <border-style> <color>? | <border-style>
border-style  = none | solid | dashed | dotted | double | rounded
border-color  = <color>
border-top    = <border-style> <color>?
border-right  = <border-style> <color>?
border-bottom = <border-style> <color>?
border-left   = <border-style> <color>?
```

**Colors:**

```
color            = <color>
background       = <color>
background-color = <color>
```

**Text:**

```
font-weight     = normal | bold
font-style      = normal | italic
text-decoration = none | underline | strikethrough
text-align      = left | center | right
vertical-align  = top | middle | bottom
white-space     = normal | nowrap | pre | pre-wrap
overflow-wrap   = normal | break-word
```

**Flexbox:**

```
flex-direction  = row | column | row-reverse | column-reverse
flex-wrap       = nowrap | wrap | wrap-reverse
flex-grow       = <number>
flex-shrink     = <number>
flex-basis      = <length> | auto
flex            = none | auto | <flex-grow> <flex-shrink>? <flex-basis>?
justify-content = flex-start | flex-end | center | space-between | space-around | space-evenly
align-items     = flex-start | flex-end | center | stretch | baseline
align-self      = auto | flex-start | flex-end | center | stretch | baseline
```

**Grid:**

```
grid-template-columns = <track-list>
grid-template-rows    = <track-list>
grid-column           = <grid-line> | <grid-line> '/' <grid-line>
grid-row              = <grid-line> | <grid-line> '/' <grid-line>
<track-list>          = <length>+
<grid-line>           = <integer> | span <integer>
```

**Gap:**

```
gap        = <length>{1,2}
row-gap    = <length>
column-gap = <length>
```

**Overflow:**

```
overflow   = visible | hidden | scroll | auto
overflow-x = visible | hidden | scroll | auto
overflow-y = visible | hidden | scroll | auto
```

**Misc:**

```
visibility = visible | hidden
z-index    = <integer>
```

**Global keywords (valid for any property):**

```
<global> = inherit | initial
```

#### Parser Implementation

TSS follows CSS's error handling model: **invalid declarations are skipped**, not rejected. A typo in one rule shouldn't break the whole stylesheet.

```rust
pub struct Parser<'a> {
    input: cssparser::Parser<'a>,
}

impl<'a> Parser<'a> {
    /// Parse a complete stylesheet
    /// Invalid rules are skipped with a warning logged
    pub fn parse_stylesheet(input: &str) -> Stylesheet;

    /// Parse a single rule
    pub fn parse_rule(&mut self) -> Result<Rule, ParseError>;

    /// Parse a selector list
    pub fn parse_selectors(&mut self) -> Result<SelectorList, ParseError>;

    /// Parse a declaration block { prop: value; ... }
    /// Invalid declarations are skipped
    pub fn parse_declarations(&mut self) -> Vec<Declaration>;

    /// Parse a single declaration
    pub fn parse_declaration(&mut self) -> Result<Declaration, ParseError>;

    /// Parse a property value based on property name
    pub fn parse_value(&mut self, property: Property) -> Result<SpecifiedValue, ParseError>;
}

/// Error recovery: skip to next semicolon or block end
fn skip_invalid_declaration(input: &mut cssparser::Parser) {
    let _ = input.parse_until_after(cssparser::Delimiter::Semicolon, |_| Ok(()));
}

/// Parse a length value: number, percentage, or calc()
fn parse_length(input: &mut cssparser::Parser) -> Result<SpecifiedLength, ParseError> {
    // Try calc()
    if input.try_parse(|i| i.expect_function_matching("calc")).is_ok() {
        return input.parse_nested_block(parse_calc_expr).map(SpecifiedLength::Calc);
    }

    // Try percentage
    if let Ok(pct) = input.try_parse(|i| i.expect_percentage()) {
        return Ok(SpecifiedLength::Percent(pct));
    }

    // Try number (with optional 'c' unit)
    let num = input.expect_number()?;
    let _ = input.try_parse(|i| i.expect_ident_matching("c")); // optional unit
    Ok(SpecifiedLength::Cells(num))
}

/// Parse a color value
fn parse_color(input: &mut cssparser::Parser) -> Result<Color, ParseError> {
    // Try hex
    if let Ok(hex) = input.try_parse(|i| i.expect_hash()) {
        return parse_hex_color(&hex);
    }

    // Try rgb()
    if input.try_parse(|i| i.expect_function_matching("rgb")).is_ok() {
        return input.parse_nested_block(parse_rgb);
    }

    // Try ansi()
    if input.try_parse(|i| i.expect_function_matching("ansi")).is_ok() {
        return input.parse_nested_block(parse_ansi);
    }

    // Try named color
    let ident = input.expect_ident()?;
    match ident.as_ref() {
        "black" => Ok(Color::Black),
        "red" => Ok(Color::Red),
        "green" => Ok(Color::Green),
        "yellow" => Ok(Color::Yellow),
        "blue" => Ok(Color::Blue),
        "magenta" => Ok(Color::Magenta),
        "cyan" => Ok(Color::Cyan),
        "white" => Ok(Color::White),
        "bright-black" => Ok(Color::BrightBlack),
        "bright-red" => Ok(Color::BrightRed),
        // ... etc
        "reset" => Ok(Color::Reset),
        _ => Err(ParseError::InvalidColor),
    }
}
```

### Units

```
/* All of these are equivalent - 20 cells */
width: 20;
width: 20c;      /* explicit cell unit */

/* Percentages work relative to parent */
width: 50%;

/* calc() works */
width: calc(100% - 10);
```

The `c` unit is optional and exists only for clarity. Bare numbers are cells.

**No other units** (`px`, `em`, `rem`, `ch`, `vw`, etc.) are supported. They have no meaning in a terminal grid.

### Properties

#### Display & Layout

```css
display: block | flex | grid | inline | none;

/* Flex properties */
flex-direction: row | column | row-reverse | column-reverse;
flex-wrap: nowrap | wrap | wrap-reverse;
flex-grow: <number>;
flex-shrink: <number>;
flex-basis: <length> | auto;
justify-content: flex-start | flex-end | center | space-between | space-around | space-evenly;
align-items: flex-start | flex-end | center | stretch | baseline;
align-self: auto | flex-start | flex-end | center | stretch | baseline;
gap: <length>;
row-gap: <length>;
column-gap: <length>;

/* Grid properties */
grid-template-columns: <track-list>;
grid-template-rows: <track-list>;
grid-column: <line> / <line>;
grid-row: <line> / <line>;

/* Positioning within parent */
align-self: ...;
justify-self: ...;
```

#### Box Model

```css
width: <length> | auto;
height: <length> | auto;
min-width: <length>;
max-width: <length> | none;
min-height: <length>;
max-height: <length> | none;

margin: <length>;           /* all sides */
margin: <length> <length>;  /* vertical horizontal */
margin: <length> <length> <length> <length>; /* top right bottom left */
margin-top: <length>;
margin-right: <length>;
margin-bottom: <length>;
margin-left: <length>;

padding: /* same syntax as margin */
padding-top: <length>;
padding-right: <length>;
padding-bottom: <length>;
padding-left: <length>;
```

#### Border

Borders in TUI are always 1 cell wide (they're drawn with box-drawing characters).

```css
border: <style> <color>;
border: <style>;
border-style: none | solid | dashed | dotted | double | rounded;
border-color: <color>;

/* Per-side (only style/color, width is always 1 or 0) */
border-top: <style> <color>;
border-right: <style> <color>;
border-bottom: <style> <color>;
border-left: <style> <color>;
```

Border styles map to box-drawing characters:

- `solid`: `─│┌┐└┘`
- `double`: `═║╔╗╚╝`
- `rounded`: `─│╭╮╰╯`
- `dashed`: `┄┆┌┐└┘` (or similar)
- `dotted`: `···`

#### Colors

```css
color: <color>;
background: <color>;
background-color: <color>;
border-color: <color>;
```

Color values:

```css
/* Named colors (basic terminal palette) */
color: black | red | green | yellow | blue | magenta | cyan | white;
color: bright-black | bright-red |...; /* bright variants */

/* ANSI 256 palette */
color: ansi(196);

/* RGB (terminals with true color support) */
color: rgb(255, 100, 50);
color: #ff6432;

/* Reset to terminal default */
color: reset;
```

#### Text

```css
font-weight: normal | bold;
font-style: normal | italic;
text-decoration: none | underline | strikethrough;

text-align: left | center | right;
vertical-align: top | middle | bottom;

/* Text wrapping */
white-space: normal | nowrap | pre | pre-wrap;
overflow-wrap: normal | break-word;
```

Note: No `font-family` or `font-size`. Terminals are monospace, one cell per character.

#### Overflow & Scrolling

```css
overflow: visible | hidden | scroll | auto;
overflow-x: ...;
overflow-y: ...;
```

`scroll` and `auto` enable scrollable regions.

#### Visibility

```css
visibility: visible | hidden;
```

#### Z-Index

```css
z-index: <integer>;
```

For controlling paint order of overlapping elements.

### Pseudo-classes

Supported pseudo-classes:

```css
:hover {
} /* Mouse is over element */
:focus {
} /* Element has keyboard focus */
:active {
} /* Element is being activated (mouse down) */
:disabled {
} /* Element is disabled */
:checked {
} /* Checkbox/radio is checked */
:first-child {
}
:last-child {
}
:nth-child(n) {
}
```

### Selectors

Full CSS selector support via the `selectors` crate:

```css
/* Element */
div {
}

/* Class */
.container {
}

/* ID */
#main {
}

/* Attribute */
[type="text"] {
}
[disabled] {
}

/* Combinators */
.parent .descendant {
}
.parent > .child {
}
.sibling + .adjacent {
}
.sibling ~ .general {
}

/* Compound */
div.foo.bar {
}
input[type="checkbox"]:checked {
}
```

### Variables

CSS custom properties work:

```css
:root {
  --primary: blue;
  --spacing: 2;
}

.button {
  background: var(--primary);
  padding: var(--spacing);
}
```

### Example Stylesheet

```css
:root {
  --bg: #1a1a2e;
  --fg: #eee;
  --accent: #e94560;
  --border: #333;
}

* {
  box-sizing: border-box;
}

body {
  background: var(--bg);
  color: var(--fg);
}

.container {
  display: flex;
  flex-direction: column;
  padding: 1;
  gap: 1;
}

.header {
  border-bottom: solid var(--border);
  padding-bottom: 1;
  font-weight: bold;
}

.button {
  display: inline;
  padding: 0 2;
  background: var(--accent);
  border: rounded var(--border);
}

.button:hover {
  background: bright-red;
}

.button:focus {
  border-color: var(--accent);
}

input {
  background: var(--bg);
  border: solid var(--border);
  padding: 0 1;
}

input:focus {
  border-color: var(--accent);
}
```

### Cascade & Inheritance

TSS implements CSS-like cascade and inheritance. It is **spec-inspired, not spec-compliant** - the mental model transfers from CSS, but edge cases and features that don't make sense for TUI are omitted.

#### Cascade

When multiple rules match an element, the cascade determines which declaration wins:

1. **Importance**: `!important` declarations beat normal declarations
2. **Origin**: Author styles beat UA (user-agent/default) styles
3. **Specificity**: Higher specificity wins (see below)
4. **Source order**: Later declarations win ties

**Specificity** is calculated as a tuple `(ids, classes, elements)`:

- `#foo` → (1, 0, 0)
- `.bar` → (0, 1, 0)
- `div` → (0, 0, 1)
- `div.bar#foo` → (1, 1, 1)
- `div .bar` → (0, 1, 1)

Higher tuples win (compared left to right). The `selectors` crate handles this for us.

**Not implemented** (intentionally):

- Cascade layers (`@layer`)
- `revert` and `revert-layer` keywords
- Complex specificity adjustments from `:is()`, `:where()`, `:not()`

#### Inheritance

Some properties **inherit** - if not explicitly set, they take their parent's value. Others **do not inherit** - they reset to their initial value.

**Inherited properties:**

- `color`
- `font-weight`
- `font-style`
- `text-decoration`
- `text-align`
- `visibility`
- `white-space`
- `overflow-wrap`

**Non-inherited properties** (everything else):

- `display`
- `width`, `height`, `min-*`, `max-*`
- `margin`, `padding`
- `border-*`
- `background`, `background-color`
- `flex-*`, `grid-*`
- `gap`, `row-gap`, `column-gap`
- `overflow`
- `z-index`
- `align-self`, `justify-self`

#### Special Keywords

```css
/* Force inheritance on non-inherited property */
background: inherit;

/* Reset to initial value */
color: initial;
```

#### Style Resolution

For each node, computed styles are resolved as follows:

```
1. Collect all rules where selector matches this node
2. Sort declarations by (importance, origin, specificity, source order)
3. For each property:
   a. If declared, use the winning declaration → "specified value"
   b. Else if property inherits, use parent's computed value
   c. Else use the property's initial value
4. Resolve relative values:
   - Percentages → resolve against parent/container
   - calc() → evaluate arithmetic
   - var() → substitute custom property values
5. Result: ComputedStyle
```

#### Custom Properties (Variables)

CSS custom properties work as expected:

```css
:root {
  --spacing: 2;
  --accent: bright-cyan;
}

.box {
  padding: var(--spacing);
  border-color: var(--accent);
}

/* Fallback values */
.other {
  color: var(--undefined, red);
}
```

Custom properties always inherit. They are substituted during style resolution before other computations.

#### Shorthand Properties

TSS supports common shorthand properties that expand to multiple longhand declarations:

**Box model shorthands:**

```css
/* margin, padding follow CSS pattern */
margin: 1; /* all sides */
margin: 1 2; /* vertical | horizontal */
margin: 1 2 3; /* top | horizontal | bottom */
margin: 1 2 3 4; /* top | right | bottom | left */

padding: 1 2; /* same pattern */
```

**Border shorthand:**

```css
border: solid red; /* style + color */
border: solid; /* style only */
border: dashed blue;

/* Per-side */
border-top: solid red;
border-right: dashed;
```

Note: No `border-width` in the shorthand - borders are always 1 cell.

**Flex shorthand:**

```css
flex: 1; /* flex-grow: 1, flex-shrink: 1, flex-basis: 0 */
flex: 1 0; /* flex-grow: 1, flex-shrink: 0, flex-basis: 0 */
flex: 1 0 auto; /* flex-grow: 1, flex-shrink: 0, flex-basis: auto */
flex: auto; /* flex-grow: 1, flex-shrink: 1, flex-basis: auto */
flex: none; /* flex-grow: 0, flex-shrink: 0, flex-basis: auto */
```

**Gap shorthand:**

```css
gap: 1; /* row-gap and column-gap */
gap: 1 2; /* row-gap | column-gap */
```

**Implementation:**

Shorthands are expanded to longhand declarations during parsing:

```rust
impl Declaration {
    /// Expand a shorthand into longhand declarations
    /// Returns None if not a shorthand
    pub fn expand_shorthand(property: &str, tokens: &[Token]) -> Option<Vec<Declaration>> {
        match property {
            "margin" => expand_box_shorthand(tokens, |v| [
                (Property::MarginTop, v[0]),
                (Property::MarginRight, v[1]),
                (Property::MarginBottom, v[2]),
                (Property::MarginLeft, v[3]),
            ]),
            "padding" => expand_box_shorthand(tokens, /* ... */),
            "border" => expand_border_shorthand(tokens),
            "flex" => expand_flex_shorthand(tokens),
            "gap" => expand_gap_shorthand(tokens),
            _ => None,
        }
    }
}

/// Expand 1/2/3/4 value pattern to 4 values
fn expand_box_values<T: Clone>(values: &[T]) -> [T; 4] {
    match values.len() {
        1 => [values[0].clone(), values[0].clone(), values[0].clone(), values[0].clone()],
        2 => [values[0].clone(), values[1].clone(), values[0].clone(), values[1].clone()],
        3 => [values[0].clone(), values[1].clone(), values[2].clone(), values[1].clone()],
        4 => [values[0].clone(), values[1].clone(), values[2].clone(), values[3].clone()],
        _ => unreachable!(),
    }
}
```

All shorthand expansions happen at parse time, so the cascade only deals with longhand properties.

#### calc()

```css
width: calc(100% - 10);
height: calc(50% + 5);
padding: calc(2 * 3);
```

Supported operators: `+`, `-`, `*`, `/`

**Not supported**: Unit conversions (there's only one unit), complex nested calc(), trigonometric functions, etc.

### DOM Integration Traits

TSS defines traits that the DOM must implement for style resolution. This decouples the style engine from the specific DOM implementation.

#### TNode

Base trait for all nodes in the tree:

```rust
pub trait TNode: Clone + Copy {
    type ConcreteElement: TElement<ConcreteNode = Self>;

    fn parent_node(&self) -> Option<Self>;
    fn first_child(&self) -> Option<Self>;
    fn last_child(&self) -> Option<Self>;
    fn prev_sibling(&self) -> Option<Self>;
    fn next_sibling(&self) -> Option<Self>;

    fn as_element(&self) -> Option<Self::ConcreteElement>;
    fn is_text_node(&self) -> bool;
    fn is_document(&self) -> bool;
}
```

#### TElement

Trait for element nodes, used by selector matching:

```rust
pub trait TElement: TNode {
    type ConcreteNode: TNode<ConcreteElement = Self>;

    /// The element's tag name (e.g., "div", "span")
    fn tag_name(&self) -> &str;

    /// The element's ID attribute, if any
    fn id(&self) -> Option<&str>;

    /// Check if element has a class
    fn has_class(&self, name: &str) -> bool;

    /// Iterate over classes
    fn classes(&self) -> impl Iterator<Item = &str>;

    /// Get an attribute value
    fn attr(&self, name: &str) -> Option<&str>;

    /// Check element state for pseudo-classes
    fn state(&self) -> ElementState;

    /// Get inline style declarations, if any
    fn inline_style(&self) -> Option<&[Declaration]>;

    /// Get or create style data storage
    fn style_data(&self) -> &StyleData;
    fn style_data_mut(&self) -> &mut StyleData;

    /// Parent element (skipping non-element nodes)
    fn parent_element(&self) -> Option<Self> {
        let mut node = self.as_node().parent_node();
        while let Some(n) = node {
            if let Some(el) = n.as_element() {
                return Some(el);
            }
            node = n.parent_node();
        }
        None
    }

    /// Previous sibling element
    fn prev_sibling_element(&self) -> Option<Self>;

    /// Next sibling element
    fn next_sibling_element(&self) -> Option<Self>;

    /// First child element
    fn first_element_child(&self) -> Option<Self>;

    /// Last child element
    fn last_element_child(&self) -> Option<Self>;

    /// Number of preceding sibling elements (for nth-child)
    fn index_in_parent(&self) -> usize;
}

/// Storage for computed styles and related data on each element
pub struct StyleData {
    /// Computed style for this element
    pub computed: Option<ComputedStyle>,
    /// Cached rule matching results (optimization)
    pub rule_cache: Option<RuleCache>,
}
```

#### TDocument

Trait for the document/root:

```rust
pub trait TDocument {
    type ConcreteNode: TNode;
    type ConcreteElement: TElement;

    fn root_element(&self) -> Option<Self::ConcreteElement>;

    fn traverse_elements(&self) -> impl Iterator<Item = Self::ConcreteElement>;
}
```

#### Stylist

The main entry point for style resolution:

```rust
pub struct Stylist {
    /// User-agent (default) stylesheets
    ua_sheets: Vec<Stylesheet>,
    /// Author stylesheets
    author_sheets: Vec<Stylesheet>,
    /// All rules, sorted for cascade
    rules: Vec<CascadedRule>,
    /// Whether rules need rebuilding
    dirty: bool,
}

struct CascadedRule {
    selector: Selector,
    declarations: Arc<[Declaration]>,
    specificity: Specificity,
    origin: Origin,
    source_order: u32,
}

#[derive(Clone, Copy, PartialEq, Eq, PartialOrd, Ord)]
pub enum Origin {
    UserAgent,
    Author,
}

impl Stylist {
    pub fn new() -> Self;

    /// Add a UA stylesheet (called once at startup)
    pub fn add_ua_stylesheet(&mut self, sheet: Stylesheet);

    /// Add an author stylesheet
    pub fn add_stylesheet(&mut self, sheet: Stylesheet);

    /// Remove an author stylesheet
    pub fn remove_stylesheet(&mut self, sheet: &Stylesheet);

    /// Clear all author stylesheets
    pub fn clear_stylesheets(&mut self);

    /// Rebuild internal rule list after stylesheet changes
    fn rebuild_rules(&mut self);

    /// Compute style for an element
    pub fn compute_style<E: TElement>(
        &self,
        element: E,
        parent_style: Option<&ComputedStyle>,
    ) -> ComputedStyle;

    /// Resolve styles for an entire tree
    pub fn resolve_tree<D: TDocument>(&self, doc: &D);
}
```

#### Selector Matching

We use the `selectors` crate for selector parsing and matching. Our `TElement` impl provides the `selectors::Element` trait:

```rust
impl<E: TElement> selectors::Element for ElementWrapper<E> {
    type Impl = KorinSelectorImpl;

    fn opaque(&self) -> selectors::OpaqueElement;
    fn parent_element(&self) -> Option<Self>;
    fn has_local_name(&self, name: &str) -> bool;
    fn has_class(&self, name: &str) -> bool;
    fn has_id(&self, id: &str) -> bool;
    fn attr_matches(&self, ...) -> bool;
    fn match_pseudo_class(&self, pc: PseudoClass) -> bool;
    // ... etc
}

/// Our selector implementation for the selectors crate
pub struct KorinSelectorImpl;

impl selectors::SelectorImpl for KorinSelectorImpl {
    type PseudoClass = PseudoClass;
    type PseudoElement = PseudoElement; // Not supported initially, but needed for the trait
    // ...
}

#[derive(Clone, Debug, PartialEq, Eq)]
pub enum PseudoClass {
    Hover,
    Focus,
    Active,
    Disabled,
    Checked,
    FirstChild,
    LastChild,
    NthChild(NthSpec),
}

/// Placeholder - pseudo-elements not supported initially
#[derive(Clone, Debug, PartialEq, Eq)]
pub enum PseudoElement {}
```

### Implementation

TSS is implemented using:

- `cssparser`: Tokenization and parsing
- `selectors`: Selector parsing and matching
- Custom cascade and property resolution

```rust
pub struct Stylesheet {
    rules: Vec<Rule>,
}

pub struct Rule {
    selectors: SelectorList,
    declarations: Vec<Declaration>,
    source_order: u32,
}

pub struct Declaration {
    property: Property,
    value: SpecifiedValue,
    important: bool,
}

pub enum Property {
    Display,
    Width,
    Height,
    // ... ~30-40 properties total
}

impl Property {
    /// Whether this property inherits by default
    pub fn inherits(&self) -> bool {
        matches!(self,
            Property::Color |
            Property::FontWeight |
            Property::FontStyle |
            Property::TextDecoration |
            Property::TextAlign |
            Property::Visibility |
            Property::WhiteSpace |
            Property::OverflowWrap
        )
    }

    /// The initial value for this property
    pub fn initial_value(&self) -> SpecifiedValue;
}

/// Value as specified in the stylesheet (may contain calc, var, percentages)
pub enum SpecifiedValue {
    /// Explicit value
    Explicit(Value),
    /// inherit keyword
    Inherit,
    /// initial keyword
    Initial,
    /// calc() expression
    Calc(CalcExpr),
    /// var() reference
    Var { name: String, fallback: Option<Box<SpecifiedValue>> },
}

/// Resolved/computed value ready for layout
pub struct ComputedStyle {
    pub display: Display,
    pub width: Dimension,
    pub height: Dimension,
    pub min_width: Dimension,
    pub max_width: Dimension,
    pub min_height: Dimension,
    pub max_height: Dimension,
    pub margin: Edges<i32>,
    pub padding: Edges<i32>,
    pub border: BorderStyle,
    pub color: Color,
    pub background: Color,
    pub font_weight: FontWeight,
    pub font_style: FontStyle,
    pub text_decoration: TextDecoration,
    pub text_align: TextAlign,
    pub vertical_align: VerticalAlign,
    pub white_space: WhiteSpace,
    pub overflow_wrap: OverflowWrap,
    pub flex_direction: FlexDirection,
    pub flex_wrap: FlexWrap,
    pub flex_grow: f32,
    pub flex_shrink: f32,
    pub flex_basis: Dimension,
    pub justify_content: JustifyContent,
    pub align_items: AlignItems,
    pub align_self: AlignSelf,
    pub gap: Size<i32>,
    pub overflow: Overflow,
    pub visibility: Visibility,
    pub z_index: i32,
}

/// Dimension that may be auto, fixed cells, or percentage
pub enum Dimension {
    Auto,
    Cells(i32),
    Percent(f32),
}
```

---

## Crate: `korin_dom`

DOM implementation with TSS for styling and Taffy for layout.

### NodeId

Slotmap-based node identifier.

```rust
slotmap::new_key_type! {
    pub struct NodeId;
}
```

### Node

```rust
pub struct Node {
    // Identity
    pub id: NodeId,

    // Tree structure
    pub parent: Option<NodeId>,
    pub children: Vec<NodeId>,

    // Content
    pub data: NodeData,

    // Element data (for elements only)
    pub tag: Option<String>,
    pub attributes: Attributes,
    pub element_data: ElementData,

    // State
    pub element_state: ElementState,  // HOVER, FOCUS, ACTIVE, etc.

    // Styling
    pub inline_style: Option<Vec<Declaration>>,
    pub computed_style: ComputedStyle,

    // Layout (Taffy)
    pub taffy_style: taffy::Style,
    pub taffy_cache: taffy::Cache,
    pub layout: taffy::Layout,

    // Scrolling
    pub scroll_offset: Point<i32>,
}

pub enum NodeData {
    Document,
    Element,
    Text(String),
}

bitflags! {
    pub struct ElementState: u8 {
        const HOVER = 1 << 0;
        const FOCUS = 1 << 1;
        const ACTIVE = 1 << 2;
        const DISABLED = 1 << 3;
        const CHECKED = 1 << 4;
    }
}
```

### ElementData

Special element handling for interactive elements.

```rust
pub enum ElementData {
    None,
    TextInput(TextInput),
    TextArea(TextArea),
    Checkbox { checked: bool },
    Select(Select),
}

impl Default for ElementData {
    fn default() -> Self {
        Self::None
    }
}

pub struct TextInput {
    pub value: String,
    pub cursor: usize,
    pub selection: Option<Range<usize>>,
}

pub struct TextArea {
    pub value: String,
    pub cursor: (usize, usize),  // (line, column)
    pub selection: Option<Range<usize>>,
}

pub struct Select {
    pub options: Vec<String>,
    pub selected: usize,
    pub open: bool,
}
```

### Document

```rust
pub struct Document {
    // Node storage
    nodes: SlotMap<NodeId, Node>,
    root: NodeId,

    // Stylesheets
    stylesheets: Vec<Stylesheet>,

    // State tracking
    hover_node: Option<NodeId>,
    focus_node: Option<NodeId>,
    active_node: Option<NodeId>,

    // Viewport
    viewport: Size<u16>,

    // Dirty tracking
    needs_restyle: HashSet<NodeId>,
    needs_layout: bool,
}

impl Document {
    pub fn new() -> Self;

    // Tree mutation
    pub fn create_element(&mut self, tag: &str) -> NodeId;
    pub fn create_text(&mut self, content: &str) -> NodeId;
    pub fn append_child(&mut self, parent: NodeId, child: NodeId);
    pub fn insert_before(&mut self, parent: NodeId, child: NodeId, reference: Option<NodeId>);
    pub fn remove(&mut self, node: NodeId);
    pub fn detach(&mut self, node: NodeId);

    // Attributes
    pub fn set_attribute(&mut self, node: NodeId, key: &str, value: &str);
    pub fn remove_attribute(&mut self, node: NodeId, key: &str);
    pub fn get_attribute(&self, node: NodeId, key: &str) -> Option<&str>;

    // Inline styles
    pub fn set_inline_style(&mut self, node: NodeId, property: &str, value: &str);

    // Text content
    pub fn set_text_content(&mut self, node: NodeId, text: &str);

    // Stylesheets
    pub fn add_stylesheet(&mut self, tss: &str);

    // Tree queries
    pub fn parent(&self, node: NodeId) -> Option<NodeId>;
    pub fn children(&self, node: NodeId) -> &[NodeId];
    pub fn root(&self) -> NodeId;
    pub fn get(&self, node: NodeId) -> Option<&Node>;
    pub fn get_mut(&mut self, node: NodeId) -> Option<&mut Node>;

    // Focus management
    pub fn focus(&mut self, node: NodeId);
    pub fn blur(&mut self);
    pub fn focus_next(&mut self) -> Option<NodeId>;
    pub fn focus_prev(&mut self) -> Option<NodeId>;
    pub fn focused(&self) -> Option<NodeId>;

    // Hover management
    pub fn set_hover(&mut self, position: Point<u16>);
    pub fn clear_hover(&mut self);

    // Layout & styling
    pub fn resolve(&mut self, viewport: Size<u16>);
    pub fn layout(&self, node: NodeId) -> &taffy::Layout;

    // Hit testing
    pub fn hit_test(&self, position: Point<u16>) -> Option<NodeId>;

    // Event dispatch
    pub fn dispatch_event(&mut self, target: NodeId, event: &Event) -> bool;
}
```

### Taffy Integration

Document implements Taffy's layout traits directly:

```rust
impl TraversePartialTree for Document { ... }
impl TraverseTree for Document { ... }
impl LayoutPartialTree for Document { ... }
impl CacheTree for Document { ... }
impl RoundTree for Document { ... }
```

### Events

```rust
pub enum Event {
    Key(KeyEvent),
    Mouse(MouseEvent),
    Focus,
    Blur,
}

pub struct KeyEvent {
    pub key: Key,
    pub modifiers: Modifiers,
}

pub struct MouseEvent {
    pub kind: MouseEventKind,
    pub position: Point<u16>,
    pub button: Option<MouseButton>,
    pub modifiers: Modifiers,
}

pub enum MouseEventKind {
    Down,
    Up,
    Move,
    Scroll { delta: i16 },
    Enter,
    Leave,
}
```

---

## Crate: `korin_reactive`

Fine-grained reactivity primitives. Forked from `leptos_reactive`.

### Signals

```rust
pub fn signal<T>(value: T) -> (ReadSignal<T>, WriteSignal<T>);
pub fn create_signal<T>(value: T) -> Signal<T>;

impl<T> Signal<T> {
    pub fn get(&self) -> T where T: Clone;
    pub fn set(&self, value: T);
    pub fn update(&self, f: impl FnOnce(&mut T));
    pub fn with<R>(&self, f: impl FnOnce(&T) -> R) -> R;
}
```

### Effects

```rust
pub fn create_effect<T>(f: impl FnMut(Option<T>) -> T);
```

### Memos

```rust
pub fn create_memo<T>(f: impl Fn(Option<&T>) -> T) -> Memo<T>;

impl<T> Memo<T> {
    pub fn get(&self) -> T where T: Clone;
    pub fn with<R>(&self, f: impl FnOnce(&T) -> R) -> R;
}
```

---

## Crate: `korin_view`

View layer for building and updating DOM.

### View Trait

```rust
pub trait View {
    type State;

    fn build(self, cursor: &mut Cursor) -> Self::State;
    fn rebuild(state: &mut Self::State, cursor: &mut Cursor);
}
```

### Mountable Trait

```rust
pub trait Mountable {
    fn unmount(&mut self, doc: &mut Document);
    fn mount(&mut self, doc: &mut Document, parent: NodeId, marker: Option<NodeId>);
}
```

### Cursor

```rust
pub struct Cursor<'a> {
    doc: &'a mut Document,
    parent: NodeId,
    current: Option<NodeId>,
}
```

### Control Flow

```rust
pub struct Show<W, F, V> {
    when: W,
    fallback: F,
    children: V,
}

pub struct Each<I, K, KF, V, VF> {
    items: I,
    key: KF,
    view: VF,
}
```

---

## Crate: `korin_macro`

### view! Macro

```rust
view! {
    <div class="container">
        <h1>"Hello, " {name}</h1>
        <button on:click=move |_| count.update(|n| *n + 1)>
            "Count: " {count}
        </button>
    </div>
}
```

### #[component] Macro

```rust
#[component]
fn Counter(initial: i32) -> impl View {
    let count = signal(initial);

    view! {
        <button on:click=move |_| count.update(|n| *n + 1)>
            "Count: " {count}
        </button>
    }
}
```

---

## Crate: `korin_render`

Rendering and event loop.

### Renderer

```rust
pub fn render(doc: &Document, frame: &mut Frame);
```

Walks the document tree and emits ratatui widgets based on:

- Layout position and size
- Computed styles (colors, borders, text attributes)
- Element content

### Text Measurement

```rust
pub fn measure_text(text: &str) -> Size<u16> {
    Size {
        width: unicode_width::UnicodeWidthStr::width(text) as u16,
        height: 1,
    }
}
```

### Event Loop

```rust
pub fn run<V: View>(root: impl FnOnce() -> V) -> Result<()> {
    // 1. Setup terminal (crossterm)
    // 2. Create document
    // 3. Build view into document
    // 4. Loop:
    //    a. Poll crossterm events
    //    b. Convert to korin events
    //    c. Dispatch to document
    //    d. Run reactive updates
    //    e. Resolve styles/layout if dirty
    //    f. Render to ratatui
}
```

---

## Dependencies

### korin_tss

```toml
[dependencies]
cssparser = "0.34"
selectors = "0.28"  # from servo
smallvec = "1"
```

### korin_dom

```toml
[dependencies]
korin_tss = { path = "../korin_tss" }
taffy = "0.7"
slotmap = "1"
bitflags = "2"
rustc-hash = "2"
```

### korin_reactive

```toml
[dependencies]
slotmap = "1"
```

### korin_view

```toml
[dependencies]
korin_dom = { path = "../korin_dom" }
korin_reactive = { path = "../korin_reactive" }
```

### korin_macro

```toml
[dependencies]
proc-macro2 = "1"
quote = "1"
syn = { version = "2", features = ["full"] }
```

### korin_render

```toml
[dependencies]
korin_dom = { path = "../korin_dom" }
ratatui = "0.29"
crossterm = "0.28"
unicode-width = "0.2"
```

### korin (umbrella)

```toml
[dependencies]
korin_tss = { path = "../korin_tss" }
korin_dom = { path = "../korin_dom" }
korin_reactive = { path = "../korin_reactive" }
korin_view = { path = "../korin_view" }
korin_macro = { path = "../korin_macro" }
korin_render = { path = "../korin_render" }
```

---

## Implementation Order

1. **korin_tss** - Parser, properties, cascade
2. **korin_dom** - Document, Node, tree ops
3. **korin_dom** - TSS integration, selector matching
4. **korin_dom** - Taffy integration, layout
5. **korin_render** - Document → ratatui
6. **korin_render** - Event loop (crossterm → events → dispatch)
7. **korin_reactive** - Fork from leptos
8. **korin_view** - View trait, Cursor, control flow
9. **korin_macro** - view!, #[component]

---

## Example

```rust
use korin::prelude::*;

static STYLES: &str = r#"
    .container {
        display: flex;
        flex-direction: column;
        padding: 1;
        gap: 1;
    }

    .title {
        font-weight: bold;
        color: bright-cyan;
    }

    .counter {
        display: flex;
        gap: 2;
        align-items: center;
    }

    button {
        padding: 0 2;
        border: rounded;
    }

    button:hover {
        background: blue;
    }

    button:focus {
        border-color: bright-blue;
    }
"#;

#[component]
fn App() -> impl View {
    let count = signal(0);

    view! {
        <div class="container">
            <h1 class="title">"Korin Counter"</h1>
            <div class="counter">
                <button on:click=move |_| count.update(|n| *n - 1)>"-"</button>
                <span>{move || count.get().to_string()}</span>
                <button on:click=move |_| count.update(|n| *n + 1)>"+"</button>
            </div>
        </div>
    }
}

fn main() -> Result<()> {
    korin::run_with_styles(STYLES, App)
}
```

---

## Future Work

- **Animations**: Simple property transitions
- **Themes**: First-class theme/color scheme support
- **DevTools**: Terminal-based DOM inspector
- **Router**: Navigation for multi-view apps
- **Hot reload**: Development-time style reloading

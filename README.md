# om.el ![Github Workflow Status](https://img.shields.io/github/workflow/status/ndwarshuis/om.el/CI)

A functional API for org-mode inspired by
[@magnars](https://github.com/magnars)'s
[dash.el](https://github.com/magnars/dash.el) and
[s.el](https://github.com/magnars/s.el) libraries.

# Installation

This package is not yet in MELPA. To install, clone this repository
somewhere into your load path:

```
git clone https://github.com/ndwarshuis/om.el
```

Then require in your emacs config:

```
(require 'om.el)
```

# Motivation

Org-mode comes with a powerful, built-in parse-tree generator
specified in `org-element.el`. The generated parse-tree is simply a
heavily-nested list which can be easily manipulated using (mostly
pure) functional code. This contrasts the majority of functions
normally used to interface with org-mode files, which are imperative
in nature (`org-insert-headine`, `outline-next-heading`, etc) as they
depend on the mutable state of Emacs buffers. In general, functional
code is
([arguably](https://en.wikipedia.org/wiki/Functional_programming#Comparison_to_imperative_programming))
more robust, readable, and testable, especially in use-cases such as
this where a stateless abstract data structure is being transformed
and queried.

The `org-element.el` provides a minimal API for handling this
parse-tree in a functional manner, but lacks many of the higher-level
functions necessary for intuitive, large-scale use. The `om.el`
package is designed to provide this API. Furthermore, it is highly
compatible with the `dash.el` package, which is a generalized
functional library for emacs-lisp.

# Org-Element Overview

Parsing a buffer with the function `org-element-parse-buffer` will
yield a parse tree composed of nodes. Nodes have types and properties
associated with them. See [the org-element API
documentation](https://orgmode.org/worg/dev/org-element-api.html#attributes)
for a list of all node types and their properties (also see the
[terminology conventions](#terminology) and [property
omissions](#properties) used in this package).

Each node is represented by a list where the first member is the type
and the second member is a plist describing the node's properties:

``` emacs-lisp
(type (:prop1 value1 :prop2 value2 ...))
```

Node types may be either leaves or branches, where branches
may have zero or more child nodes and leaves may not have child nodes
at all. Leaves will always have lists of the form shown above.
Branches, on the other hand, have their children appended to the end:

``` emacs-lisp
(type (:prop1 value1 :prop2 value2) child1 child2 ...)
```

In addition to leaves and branches, node types can belong to one of
two classes:
- Objects: roughly correspond to raw, possibly-formatted text
- Elements: more complex structures which may be built from objects

Within the branch node types, there are restrictions of which class
is allowed to be a child depending on the type. There are three of
these restrictions:
- Branch element with child elements (aka 'greater elements'): these
  are element types that are generally nestable inside one another (eg
  headlines, plain-lists, items)
- Branch elements with child objects (aka 'object containers'): these
  are element types that hold textual information (eg paragraph)
- Branch objects with child objects (aka 'recursive objects'): these
  are object types used primarily for text formating (bold, italic,
  underline, etc)

Note: it is never allowed for an element type to be a child of a
branch object type.
      
# Conventions

## Terminology

This package takes several deviations from the original terminology
found in `org-element.el`. Generally, the names used here conform to
terminology more often used with tree data structures:
- 'node' is used here to describe a vertex in the parse
  tree, where 'element' and 'object' are two classes used to describe
  said vertex (`org-element.el` seems to use 'element' to generally
  mean 'node' and uses 'object' to further specify)
- 'child' and 'children' are used here instead of 'content' and
  'contents'
- 'branch' is used here instead of 'container'. Furthermore, 'leaf' is
  used to describe the converse of 'branch' (there does not seem to be
  an equivalent term in `org-element.el`)
- `org-element.el` uses 'attribute(s)' and 'property(ies)'
  interchangably to describe nodes; here only 'property(ies)' is used

## Properties

The properties `:begin`, `:end`, `:contents-begin`, `:contents-end`,
and `post-affiliated` are not exposed by this API. They are not
necessary for manipulating the functional representation of the parse
tree. In addition to these, some properties unique to certain types
are not exposed for the same reason. Each type's build function
describes the properties that are available.

## Threading

Each function that operates on an element/object will take the
element/object as its right-most argument. This allows convenient
function chaining using `dash.el`'s right-threading operators (`->>`
and `-some->>`). The examples below almost exclusively demonstrate
this pattern. Additionally, the right-argument convention also allows
convenient partial application using `-partial` from `dash.el`.

## Higher-order functions

Higher-order functions (functions that take other functions as
arguments) have two forms. The first takes a (usually unary) function
and applies it:

``` emacs-lisp
(om-map-property :value (lambda (s) (concat "foo" s)) node)
(om-map-property :value (-partial concat "foo") node)
```

This can equivalently be written using an anaphoric form where the
original function name is appended with `*`. The symbol `it`
carries the value of the unary argument (unless otherwise specified):

``` emacs-lisp
(om-map-property* :value (concat "foo" it) node)
```

## Side effect functions

All functions that read and write from buffers are named like
`om-OPERATION-THING-at` where `OPERATION` is some operation to be
performed on `THING` in the current buffer. All these functions take
`point` as one of their arguments to denote where in the buffer to
perform `OPERATION`.

All of these functions have current-point convenience analogues that
are named as `om-OPERATION-this-THING` where `OPERATION` and `THING`
carry the same meaning, but `OPERATION` is done at the current point
and `point` is not an argument to the function.

For the sake of brevity, only the former form of these functions are
given in the examples below.

# Function Summary


## String Conversion


Convert nodes to strings.

* [om-to-string](#om-to-string-node) `(node)`
* [om-to-trimmed-string](#om-to-trimmed-string-node) `(node)`

## Buffer Parsing


Parse buffers to trees.

* [om-parse-object-at](#om-parse-object-at-point) `(point)`
* [om-parse-element-at](#om-parse-element-at-point) `(point)`
* [om-parse-table-row-at](#om-parse-table-row-at-point) `(point)`
* [om-parse-headline-at](#om-parse-headline-at-point) `(point)`
* [om-parse-subtree-at](#om-parse-subtree-at-point) `(point)`
* [om-parse-item-at](#om-parse-item-at-point) `(point)`
* [om-parse-section-at](#om-parse-section-at-point) `(point)`

## Building


Build new nodes.


### Leaf Object Nodes

* [om-build-code](#om-build-code-value-key-post-blank) `(value &key post-blank)`
* [om-build-entity](#om-build-entity-name-key-use-brackets-p-post-blank) `(name &key use-brackets-p post-blank)`
* [om-build-export-snippet](#om-build-export-snippet-back-end-value-key-post-blank) `(back-end value &key post-blank)`
* [om-build-inline-babel-call](#om-build-inline-babel-call-call-key-inside-header-arguments-end-header-post-blank) `(call &key inside-header arguments end-header post-blank)`
* [om-build-inline-src-block](#om-build-inline-src-block-language-key-parameters-value--post-blank) `(language &key parameters (value "") post-blank)`
* [om-build-line-break](#om-build-line-break-key-post-blank) `(&key post-blank)`
* [om-build-latex-fragment](#om-build-latex-fragment-value-key-post-blank) `(value &key post-blank)`
* [om-build-macro](#om-build-macro-key-key-args-post-blank) `(key &key args post-blank)`
* [om-build-statistics-cookie](#om-build-statistics-cookie-value-key-post-blank) `(value &key post-blank)`
* [om-build-target](#om-build-target-value-key-post-blank) `(value &key post-blank)`
* [om-build-timestamp](#om-build-timestamp-type-year-start-month-start-day-start-year-end-month-end-day-end-key-hour-start-minute-start-hour-end-minute-end-repeater-type-repeater-unit-repeater-value-warning-type-warning-unit-warning-value-post-blank) `(type year-start month-start day-start year-end month-end day-end &key hour-start minute-start hour-end minute-end repeater-type repeater-unit repeater-value warning-type warning-unit warning-value post-blank)`
* [om-build-verbatim](#om-build-verbatim-value-key-post-blank) `(value &key post-blank)`

### Branch Object Nodes

* [om-build-bold](#om-build-bold-key-post-blank-rest-object-nodes) `(&key post-blank &rest object-nodes)`
* [om-build-footnote-reference](#om-build-footnote-reference-key-label-post-blank-rest-object-nodes) `(&key label post-blank &rest object-nodes)`
* [om-build-italic](#om-build-italic-key-post-blank-rest-object-nodes) `(&key post-blank &rest object-nodes)`
* [om-build-link](#om-build-link-path-key-format-type-fuzzy-post-blank-rest-object-nodes) `(path &key format (type "fuzzy") post-blank &rest object-nodes)`
* [om-build-radio-target](#om-build-radio-target-key-post-blank-rest-object-nodes) `(&key post-blank &rest object-nodes)`
* [om-build-strike-through](#om-build-strike-through-key-post-blank-rest-object-nodes) `(&key post-blank &rest object-nodes)`
* [om-build-superscript](#om-build-superscript-key-use-brackets-p-post-blank-rest-object-nodes) `(&key use-brackets-p post-blank &rest object-nodes)`
* [om-build-subscript](#om-build-subscript-key-use-brackets-p-post-blank-rest-object-nodes) `(&key use-brackets-p post-blank &rest object-nodes)`
* [om-build-table-cell](#om-build-table-cell-key-post-blank-rest-object-nodes) `(&key post-blank &rest object-nodes)`
* [om-build-underline](#om-build-underline-key-post-blank-rest-object-nodes) `(&key post-blank &rest object-nodes)`

### Leaf Element Nodes

* [om-build-babel-call](#om-build-babel-call-call-key-inside-header-arguments-end-header-post-blank) `(call &key inside-header arguments end-header post-blank)`
* [om-build-clock](#om-build-clock-value-key-post-blank) `(value &key post-blank)`
* [om-build-comment](#om-build-comment-value-key-post-blank) `(value &key post-blank)`
* [om-build-comment-block](#om-build-comment-block-key-value--post-blank) `(&key (value "") post-blank)`
* [om-build-diary-sexp](#om-build-diary-sexp-key-value-post-blank) `(&key value post-blank)`
* [om-build-example-block](#om-build-example-block-key-preserve-indent-switches-value--post-blank) `(&key preserve-indent switches (value "") post-blank)`
* [om-build-export-block](#om-build-export-block-type-value-key-post-blank) `(type value &key post-blank)`
* [om-build-fixed-width](#om-build-fixed-width-value-key-post-blank) `(value &key post-blank)`
* [om-build-horizontal-rule](#om-build-horizontal-rule-key-post-blank) `(&key post-blank)`
* [om-build-keyword](#om-build-keyword-key-value-key-post-blank) `(key value &key post-blank)`
* [om-build-latex-environment](#om-build-latex-environment-value-key-post-blank) `(value &key post-blank)`
* [om-build-node-property](#om-build-node-property-key-value-key-post-blank) `(key value &key post-blank)`
* [om-build-planning](#om-build-planning-key-closed-deadline-scheduled-post-blank) `(&key closed deadline scheduled post-blank)`
* [om-build-src-block](#om-build-src-block-key-value--language-parameters-preserve-indent-switches-post-blank) `(&key (value "") language parameters preserve-indent switches post-blank)`

### Branch Element Nodes with Child Object Nodes

* [om-build-paragraph](#om-build-paragraph-key-post-blank-rest-object-nodes) `(&key post-blank &rest object-nodes)`
* [om-build-table-row](#om-build-table-row-key-post-blank-rest-object-nodes) `(&key post-blank &rest object-nodes)`
* [om-build-verse-block](#om-build-verse-block-key-post-blank-rest-object-nodes) `(&key post-blank &rest object-nodes)`

### Branch Element Nodes with Child Element Nodes

* [om-build-center-block](#om-build-center-block-key-post-blank-rest-element-nodes) `(&key post-blank &rest element-nodes)`
* [om-build-drawer](#om-build-drawer-drawer-name-key-post-blank-rest-element-nodes) `(drawer-name &key post-blank &rest element-nodes)`
* [om-build-dynamic-block](#om-build-dynamic-block-block-name-key-arguments-post-blank-rest-element-nodes) `(block-name &key arguments post-blank &rest element-nodes)`
* [om-build-footnote-definition](#om-build-footnote-definition-label-key-post-blank-rest-element-nodes) `(label &key post-blank &rest element-nodes)`
* [om-build-headline](#om-build-headline-key-archivedp-commentedp-footnote-section-p-level-1-pre-blank-0-priority-tags-title-todo-keyword-post-blank-rest-element-nodes) `(&key archivedp commentedp footnote-section-p (level 1) (pre-blank 0) priority tags title todo-keyword post-blank &rest element-nodes)`
* [om-build-item](#om-build-item-key-bullet-quote---checkbox-counter-tag-post-blank-rest-element-nodes) `(&key (bullet '-) checkbox counter tag post-blank &rest element-nodes)`
* [om-build-plain-list](#om-build-plain-list-key-post-blank-rest-element-nodes) `(&key post-blank &rest element-nodes)`
* [om-build-property-drawer](#om-build-property-drawer-key-post-blank-rest-element-nodes) `(&key post-blank &rest element-nodes)`
* [om-build-quote-block](#om-build-quote-block-key-post-blank-rest-element-nodes) `(&key post-blank &rest element-nodes)`
* [om-build-section](#om-build-section-key-post-blank-rest-element-nodes) `(&key post-blank &rest element-nodes)`
* [om-build-special-block](#om-build-special-block-type-key-post-blank-rest-element-nodes) `(type &key post-blank &rest element-nodes)`
* [om-build-table](#om-build-table-key-tblfm-post-blank-rest-element-nodes) `(&key tblfm post-blank &rest element-nodes)`

### Miscellaneous Builders

* [om-build-secondary-string!](#om-build-secondary-string-string) `(string)`
* [om-build-table-row-hline](#om-build-table-row-hline-key-post-blank) `(&key post-blank)`
* [om-build-timestamp-diary](#om-build-timestamp-diary-form-key-post-blank) `(form &key post-blank)`

### Shorthand Builders


Build nodes with more convenient/shorter syntax.

* [om-build-timestamp!](#om-build-timestamp-start-key-end-active-repeater-warning-post-blank) `(start &key end active repeater warning post-blank)`
* [om-build-clock!](#om-build-clock-start-key-end-post-blank) `(start &key end post-blank)`
* [om-build-planning!](#om-build-planning-key-closed-deadline-scheduled-post-blank) `(&key closed deadline scheduled post-blank)`
* [om-build-property-drawer!](#om-build-property-drawer-key-post-blank-rest-keyvals) `(&key post-blank &rest keyvals)`
* [om-build-headline!](#om-build-headline-key-level-1-title-text-todo-keyword-tags-pre-blank-priority-commentedp-archivedp-post-blank-planning-statistics-cookie-section-children-rest-subheadlines) `(&key (level 1) title-text todo-keyword tags pre-blank priority commentedp archivedp post-blank planning statistics-cookie section-children &rest subheadlines)`
* [om-build-item!](#om-build-item-key-post-blank-bullet-checkbox-tag-paragraph-counter-rest-children) `(&key post-blank bullet checkbox tag paragraph counter &rest children)`
* [om-build-paragraph!](#om-build-paragraph-string-key-post-blank) `(string &key post-blank)`
* [om-build-table-cell!](#om-build-table-cell-string) `(string)`
* [om-build-table-row!](#om-build-table-row-row-list) `(row-list)`
* [om-build-table!](#om-build-table-key-tblfm-post-blank-rest-row-lists) `(&key tblfm post-blank &rest row-lists)`

## Type Predicates


Test node types.

* [om-get-type](#om-get-type-node) `(node)`
* [om-is-type](#om-is-type-type-node) `(type node)`
* [om-is-any-type](#om-is-any-type-types-node) `(types node)`
* [om-is-element](#om-is-element-node) `(node)`
* [om-is-branch-node](#om-is-branch-node-node) `(node)`
* [om-node-may-have-child-objects](#om-node-may-have-child-objects-node) `(node)`
* [om-node-may-have-child-elements](#om-node-may-have-child-elements-node) `(node)`

## Property Manipulation


Set, get, and map properties of nodes.


### Generic

* [om-contains-point-p](#om-contains-point-p-point-node) `(point node)`
* [om-set-property](#om-set-property-prop-value-node) `(prop value node)`
* [om-set-properties](#om-set-properties-plist-node) `(plist node)`
* [om-get-property](#om-get-property-prop-node) `(prop node)`
* [om-map-property](#om-map-property-prop-fun-node) `(prop fun node)`
* [om-map-properties](#om-map-properties-plist-node) `(plist node)`
* [om-toggle-property](#om-toggle-property-prop-node) `(prop node)`
* [om-shift-property](#om-shift-property-prop-n-node) `(prop n node)`
* [om-insert-into-property](#om-insert-into-property-prop-index-string-node) `(prop index string node)`
* [om-remove-from-property](#om-remove-from-property-prop-string-node) `(prop string node)`
* [om-plist-put-property](#om-plist-put-property-prop-key-value-node) `(prop key value node)`
* [om-plist-remove-property](#om-plist-remove-property-prop-key-node) `(prop key node)`

### Clock

* [om-clock-is-running](#om-clock-is-running-clock) `(clock)`

### Entity

* [om-entity-get-replacement](#om-entity-get-replacement-key-entity) `(key entity)`

### Headline

* [om-headline-set-title!](#om-headline-set-title-title-text-stats-cookie-value-headline) `(title-text stats-cookie-value headline)`
* [om-headline-is-done](#om-headline-is-done-headline) `(headline)`
* [om-headline-has-tag](#om-headline-has-tag-tag-headline) `(tag headline)`
* [om-headline-get-statistics-cookie](#om-headline-get-statistics-cookie-headline) `(headline)`

### Item

* [om-item-toggle-checkbox](#om-item-toggle-checkbox-item) `(item)`

### Planning

* [om-planning-set-timestamp!](#om-planning-set-timestamp-prop-planning-list-planning) `(prop planning-list planning)`

### Statistics Cookie

* [om-statistics-cookie-is-complete](#om-statistics-cookie-is-complete-statistics-cookie) `(statistics-cookie)`

### Timestamp (Auxiliary)


Functions to work with timestamp data

* [om-time-to-unixtime](#om-time-to-unixtime-time) `(time)`
* [om-unixtime-to-time-long](#om-unixtime-to-time-long-unixtime) `(unixtime)`
* [om-unixtime-to-time-short](#om-unixtime-to-time-short-unixtime) `(unixtime)`

### Timestamp (Standard)

* [om-timestamp-get-start-time](#om-timestamp-get-start-time-timestamp) `(timestamp)`
* [om-timestamp-get-end-time](#om-timestamp-get-end-time-timestamp) `(timestamp)`
* [om-timestamp-get-range](#om-timestamp-get-range-timestamp) `(timestamp)`
* [om-timestamp-is-active](#om-timestamp-is-active-timestamp) `(timestamp)`
* [om-timestamp-is-ranged](#om-timestamp-is-ranged-timestamp) `(timestamp)`
* [om-timestamp-range-contains-p](#om-timestamp-range-contains-p-unixtime-timestamp) `(unixtime timestamp)`
* [om-timestamp-set-collapsed](#om-timestamp-set-collapsed-flag-timestamp) `(flag timestamp)`
* [om-timestamp-set-start-time](#om-timestamp-set-start-time-time-timestamp) `(time timestamp)`
* [om-timestamp-set-end-time](#om-timestamp-set-end-time-time-timestamp) `(time timestamp)`
* [om-timestamp-set-single-time](#om-timestamp-set-single-time-time-timestamp) `(time timestamp)`
* [om-timestamp-set-double-time](#om-timestamp-set-double-time-time1-time2-timestamp) `(time1 time2 timestamp)`
* [om-timestamp-set-range](#om-timestamp-set-range-range-timestamp) `(range timestamp)`
* [om-timestamp-set-active](#om-timestamp-set-active-flag-timestamp) `(flag timestamp)`
* [om-timestamp-shift](#om-timestamp-shift-n-unit-timestamp) `(n unit timestamp)`
* [om-timestamp-shift-start](#om-timestamp-shift-start-n-unit-timestamp) `(n unit timestamp)`
* [om-timestamp-shift-end](#om-timestamp-shift-end-n-unit-timestamp) `(n unit timestamp)`
* [om-timestamp-toggle-active](#om-timestamp-toggle-active-timestamp) `(timestamp)`
* [om-timestamp-truncate](#om-timestamp-truncate-timestamp) `(timestamp)`
* [om-timestamp-truncate-start](#om-timestamp-truncate-start-timestamp) `(timestamp)`
* [om-timestamp-truncate-end](#om-timestamp-truncate-end-timestamp) `(timestamp)`

### Timestamp (diary)

* [om-timestamp-diary-set-value](#om-timestamp-diary-set-value-form-timestamp-diary) `(form timestamp-diary)`

## Branch/Child Manipulation


Set, get, and map the children of branch nodes.


### Polymorphic

* [om-children-contain-point](#om-children-contain-point-point-branch-node) `(point branch-node)`
* [om-get-children](#om-get-children-branch-node) `(branch-node)`
* [om-set-children](#om-set-children-children-branch-node) `(children branch-node)`
* [om-map-children](#om-map-children-fun-branch-node) `(fun branch-node)`
* [om-is-childless](#om-is-childless-branch-node) `(branch-node)`

### Object Nodes

* [om-unwrap](#om-unwrap-object-node) `(object-node)`
* [om-unwrap-types-deep](#om-unwrap-types-deep-types-object-node) `(types object-node)`
* [om-unwrap-deep](#om-unwrap-deep-object-node) `(object-node)`

### Secondary Strings

* [om-flatten](#om-flatten-secondary-string) `(secondary-string)`
* [om-flatten-types-deep](#om-flatten-types-deep-types-secondary-string) `(types secondary-string)`
* [om-flatten-deep](#om-flatten-deep-secondary-string) `(secondary-string)`

### Headline

* [om-headline-get-node-properties](#om-headline-get-node-properties-headline) `(headline)`
* [om-headline-get-properties-drawer](#om-headline-get-properties-drawer-headline) `(headline)`
* [om-headline-get-planning](#om-headline-get-planning-headline) `(headline)`
* [om-headline-set-planning](#om-headline-set-planning-planning-headline) `(planning headline)`
* [om-headline-get-subheadlines](#om-headline-get-subheadlines-headline) `(headline)`
* [om-headline-get-section](#om-headline-get-section-headline) `(headline)`
* [om-headline-get-path](#om-headline-get-path-headline) `(headline)`
* [om-headline-update-item-statistics](#om-headline-update-item-statistics-headline) `(headline)`
* [om-headline-update-todo-statistics](#om-headline-update-todo-statistics-headline) `(headline)`
* [om-headline-indent-subheadline](#om-headline-indent-subheadline-index-headline) `(index headline)`
* [om-headline-indent-subtree](#om-headline-indent-subtree-index-headline) `(index headline)`
* [om-headline-unindent-subheadline](#om-headline-unindent-subheadline-index-child-index-headline) `(index child-index headline)`
* [om-headline-unindent-all-subheadlines](#om-headline-unindent-all-subheadlines-index-headline) `(index headline)`

### Plain List

* [om-plain-list-set-type](#om-plain-list-set-type-type-plain-list) `(type plain-list)`
* [om-plain-list-indent-item](#om-plain-list-indent-item-index-plain-list) `(index plain-list)`
* [om-plain-list-indent-item-tree](#om-plain-list-indent-item-tree-index-plain-list) `(index plain-list)`
* [om-plain-list-unindent-item](#om-plain-list-unindent-item-index-child-index-plain-list) `(index child-index plain-list)`
* [om-plain-list-unindent-all-items](#om-plain-list-unindent-all-items-index-plain-list) `(index plain-list)`

### Table

* [om-table-get-cell](#om-table-get-cell-row-index-column-index-table) `(row-index column-index table)`
* [om-table-delete-column](#om-table-delete-column-column-index-table) `(column-index table)`
* [om-table-delete-row](#om-table-delete-row-row-index-table) `(row-index table)`
* [om-table-insert-column!](#om-table-insert-column-column-index-column-text-table) `(column-index column-text table)`
* [om-table-insert-row!](#om-table-insert-row-row-index-row-text-table) `(row-index row-text table)`
* [om-table-replace-cell!](#om-table-replace-cell-row-index-column-index-cell-text-table) `(row-index column-index cell-text table)`
* [om-table-replace-column!](#om-table-replace-column-column-index-column-text-table) `(column-index column-text table)`
* [om-table-replace-row!](#om-table-replace-row-row-index-row-text-table) `(row-index row-text table)`

## Node Matching


Use pattern-matching to selectively perform operations on nodes in trees.

* [om-match](#om-match-pattern-node) `(pattern node)`
* [om-match-delete](#om-match-delete-pattern-node) `(pattern node)`
* [om-match-extract](#om-match-extract-pattern-node) `(pattern node)`
* [om-match-map](#om-match-map-pattern-fun-node) `(pattern fun node)`
* [om-match-mapcat](#om-match-mapcat-pattern-fun-node) `(pattern fun node)`
* [om-match-replace](#om-match-replace-pattern-node-node) `(pattern node* node)`
* [om-match-insert-before](#om-match-insert-before-pattern-node-node) `(pattern node* node)`
* [om-match-insert-after](#om-match-insert-after-pattern-node-node) `(pattern node* node)`
* [om-match-insert-within](#om-match-insert-within-pattern-index-node-node) `(pattern index node* node)`
* [om-match-splice](#om-match-splice-pattern-nodes-node) `(pattern nodes* node)`
* [om-match-splice-before](#om-match-splice-before-pattern-nodes-node) `(pattern nodes* node)`
* [om-match-splice-after](#om-match-splice-after-pattern-nodes-node) `(pattern nodes* node)`
* [om-match-splice-within](#om-match-splice-within-pattern-index-nodes-node) `(pattern index nodes* node)`
* [om-match-do](#om-match-do-pattern-fun-node) `(pattern fun node)`

## Buffer Side Effects


Map node manipulations into buffers.


### Insert

* [om-insert](#om-insert-point-node) `(point node)`
* [om-insert-tail](#om-insert-tail-point-node) `(point node)`

### Update

* [om-update](#om-update-fun-node) `(fun node)`
* [om-update-object-at](#om-update-object-at-point-fun) `(point fun)`
* [om-update-element-at](#om-update-element-at-point-fun) `(point fun)`
* [om-update-table-row-at](#om-update-table-row-at-point-fun) `(point fun)`
* [om-update-item-at](#om-update-item-at-point-fun) `(point fun)`
* [om-update-headline-at](#om-update-headline-at-point-fun) `(point fun)`
* [om-update-subtree-at](#om-update-subtree-at-point-fun) `(point fun)`
* [om-update-section-at](#om-update-section-at-point-fun) `(point fun)`

### Misc

* [om-fold](#om-fold-node) `(node)`
* [om-unfold](#om-unfold-node) `(node)`

# Function Examples


## String Conversion


Convert nodes to strings.

#### om-to-string `(node)`

Return **`node`** as an interpreted string without text properties.

```el
(om-to-string '(bold (:begin 1 :end 5 :parent nil :post-blank 0 :post-affiliated nil)
			   "text"))
 ;; => "*text*"

(om-to-string '(bold (:begin 1 :end 5 :parent nil :post-blank 3 :post-affiliated nil)
			   "text"))
 ;; => "*text*   "

(om-to-string nil)
 ;; => ""

```

#### om-to-trimmed-string `(node)`

Like [`om-to-string`](#om-to-string-node) but strip whitespace when returning **`node`**.

```el
(om-to-trimmed-string '(bold (:begin 1 :end 5 :parent nil :post-blank 0 :post-affiliated nil)
				   "text"))
 ;; => "*text*"

(om-to-trimmed-string '(bold (:begin 1 :end 5 :parent nil :post-blank 3 :post-affiliated nil)
				   "text"))
 ;; => "*text*"

(om-to-trimmed-string nil)
 ;; => ""

```


## Buffer Parsing


Parse buffers to trees.

#### om-parse-object-at `(point)`

Return object node under **`point`** or nil if not on an object.

```el
;; Given the following contents:
; *text*

(->> (om-parse-object-at 1)
     (car))
 ;; => 'bold

;; Given the following contents:
; [2019-01-01 Tue]

(->> (om-parse-object-at 1)
     (car))
 ;; => 'timestamp

;; Given the following contents:
; - notme

;; Return nil when parsing an element
(om-parse-object-at
 1)
 ;; => nil

```

#### om-parse-element-at `(point)`

Return element node under **`point`** or nil if not on an element.

This function will return every element available in `om-elements`
with the exception of `section`, `item`, and `table-row`. To
specifically parse these, use the functions [`om-parse-section-at`](#om-parse-section-at-point),
[`om-parse-item-at`](#om-parse-item-at-point), and [`om-parse-table-row-at`](#om-parse-table-row-at-point).

```el
;; Given the following contents:
; #+CALL: ktulu()

(->> (om-parse-element-at 1)
     (car))
 ;; => 'babel-call

;; Given the following contents:
; - plain-list

;; Give the plain-list, not the item for this function
(->> (om-parse-element-at 1)
     (car))
 ;; => 'plain-list

;; Given the following contents:
; | R | A |
; | G | E |

;; Return a table, not the table-row for this function
(->> (om-parse-element-at 1)
     (car))
 ;; => 'table

```

#### om-parse-table-row-at `(point)`

Return table-row node under **`point`** or nil if not on a table-row.

```el
;; Given the following contents:
; | bow | stroke |

;; Return the row itself
(->> (om-parse-table-row-at 1)
     (car))
 ;; => 'table-row

;; Also return the row when not at beginning of line
(->> (om-parse-table-row-at 5)
     (car))
 ;; => 'table-row

;; Given the following contents:
; - bow and arrow choke

;; Return nil if not a table-row
(->> (om-parse-table-row-at 1)
     (car))
 ;; => nil

```

#### om-parse-headline-at `(point)`

Return headline node under **`point`** or nil if not on a headline.
**`point`** does not need to be on the headline itself. Only the headline
and its section will be returned. To include subheadlines, use
[`om-parse-subtree-at`](#om-parse-subtree-at-point).

```el
;; Given the following contents:
; * headline

;; Return the headline itself
(->> (om-parse-headline-at 1)
     (om-to-trimmed-string))
 ;; => "* headline"

;; Given the following contents:
; * headline
; section crap

;; Return headline and section
(->> (om-parse-headline-at 1)
     (om-to-trimmed-string))
 ;; => "* headline
 ;      section crap"

;; Return headline when point is in the section
(->> (om-parse-headline-at 12)
     (om-to-trimmed-string))
 ;; => "* headline
 ;      section crap"

;; Given the following contents:
; * headline
; section crap
; ** not parsed

;; Don't parse any subheadlines
(->> (om-parse-headline-at 1)
     (om-to-trimmed-string))
 ;; => "* headline
 ;      section crap"

;; Given the following contents:
; nothing nowhere

;; Return nil if not under a headline
(->> (om-parse-headline-at 1)
     (om-to-trimmed-string))
 ;; => ""

```

#### om-parse-subtree-at `(point)`

Return headline node under **`point`** or nil if not on a headline.
**`point`** does not need to be on the headline itself. Unlike
[`om-parse-headline-at`](#om-parse-headline-at-point), the returned node will include
child headlines.

```el
;; Given the following contents:
; * headline

;; Return the headline itself
(->> (om-parse-subtree-at 1)
     (om-to-trimmed-string))
 ;; => "* headline"

;; Given the following contents:
; * headline
; section crap

;; Return headline and section
(->> (om-parse-subtree-at 1)
     (om-to-trimmed-string))
 ;; => "* headline
 ;      section crap"

;; Return headline when point is in the section
(->> (om-parse-subtree-at 12)
     (om-to-trimmed-string))
 ;; => "* headline
 ;      section crap"

;; Given the following contents:
; * headline
; section crap
; ** parsed

;; Return all the subheadlines
(->> (om-parse-subtree-at 1)
     (om-to-trimmed-string))
 ;; => "* headline
 ;      section crap
 ;      ** parsed"

;; Given the following contents:
; nothing nowhere

;; Return nil if not under a headline
(->> (om-parse-subtree-at 1)
     (om-to-trimmed-string))
 ;; => ""

```

#### om-parse-item-at `(point)`

Return item node under **`point`** or nil if not on an item.
This will return the item node even if **`point`** is not at the beginning
of the line.

```el
;; Given the following contents:
; - item

;; Return the item itself
(->> (om-parse-item-at 1)
     (om-to-trimmed-string))
 ;; => "- item"

;; Also return the item when not at beginning of line
(->> (om-parse-item-at 5)
     (om-to-trimmed-string))
 ;; => "- item"

;; Given the following contents:
; - item
;   - item 2

;; Return item and its subitems
(->> (om-parse-item-at 1)
     (om-to-trimmed-string))
 ;; => "- item
 ;        - item 2"

;; Given the following contents:
; * not item

;; Return nil if not an item
(->> (om-parse-item-at 1)
     (om-to-trimmed-string))
 ;; => ""

```

#### om-parse-section-at `(point)`

Return section node under **`point`** or nil if not on a section.
If **`point`** is on or within a headline, return the section under that
headline. If **`point`** is before the first headline (if any), return
the section at the top of the org buffer.

```el
;; Given the following contents:
; over headline
; * headline
; under headline

;; Return the section above the headline
(->> (om-parse-section-at 1)
     (om-to-trimmed-string))
 ;; => "over headline"

;; Return the section under headline
(->> (om-parse-section-at 25)
     (om-to-trimmed-string))
 ;; => "under headline"

;; Given the following contents:
; * headline
; ** subheadline

;; Return nil if no section under headline
(->> (om-parse-section-at 1)
     (om-to-trimmed-string))
 ;; => ""

;; Given the following contents:
; 

;; Return nil if no section at all
(->> (om-parse-section-at 1)
     (om-to-trimmed-string))
 ;; => ""

```


## Building


Build new nodes.


### Leaf Object Nodes

#### om-build-code `(value &key post-blank)`

Build a code object node.

The following properties are settable:
- **`value`**: (required) a string
- **`post-blank`**: a non-negative integer

```el
(->> (om-build-code "text")
     (om-to-string))
 ;; => "~text~"

```

#### om-build-entity `(name &key use-brackets-p post-blank)`

Build an entity object node.

The following properties are settable:
- **`name`**: (required) a string that makes `org-entity-get` return non-nil
- **`use-brackets-p`**:  nil or t
- **`post-blank`**: a non-negative integer

```el
(->> (om-build-entity "gamma")
     (om-to-string))
 ;; => "\\gamma"

```

#### om-build-export-snippet `(back-end value &key post-blank)`

Build an export-snippet object node.

The following properties are settable:
- **`back-end`**: (required) a oneline string
- **`value`**: (required) a string
- **`post-blank`**: a non-negative integer

```el
(->> (om-build-export-snippet "back" "value")
     (om-to-string))
 ;; => "@@back:value@@"

```

#### om-build-inline-babel-call `(call &key inside-header arguments end-header post-blank)`

Build an inline-babel-call object node.

The following properties are settable:
- **`call`**: (required) a oneline string
- **`inside-header`**:  a plist
- **`arguments`**:  a list of oneline strings
- **`end-header`**:  a plist
- **`post-blank`**: a non-negative integer

```el
(->> (om-build-inline-babel-call "name")
     (om-to-string))
 ;; => "call_name()"

(->> (om-build-inline-babel-call "name" :arguments '("n=4"))
     (om-to-string))
 ;; => "call_name(n=4)"

(->> (om-build-inline-babel-call "name" :inside-header '(:key val))
     (om-to-string))
 ;; => "call_name[:key val]()"

(->> (om-build-inline-babel-call "name" :end-header '(:key val))
     (om-to-string))
 ;; => "call_name()[:key val]"

```

#### om-build-inline-src-block `(language &key parameters (value "") post-blank)`

Build an inline-src-block object node.

The following properties are settable:
- **`language`**: (required) a oneline string
- **`parameters`**:  a plist
- **`value`**: (default `""`) a string
- **`post-blank`**: a non-negative integer

```el
(->> (om-build-inline-src-block "lang")
     (om-to-string))
 ;; => "src_lang{}"

(->> (om-build-inline-src-block "lang" :value "value")
     (om-to-string))
 ;; => "src_lang{value}"

(->> (om-build-inline-src-block "lang" :value "value" :parameters '(:key val))
     (om-to-string))
 ;; => "src_lang[:key val]{value}"

```

#### om-build-line-break `(&key post-blank)`

Build a line-break object node.

The following properties are settable:

- **`post-blank`**: a non-negative integer

```el
(->> (om-build-line-break)
     (om-to-string))
 ;; => "\\\\
 ;      "

```

#### om-build-latex-fragment `(value &key post-blank)`

Build a latex-fragment object node.

The following properties are settable:
- **`value`**: (required) a string
- **`post-blank`**: a non-negative integer

```el
(->> (om-build-latex-fragment "$2+2=5$")
     (om-to-string))
 ;; => "$2+2=5$"

```

#### om-build-macro `(key &key args post-blank)`

Build a macro object node.

The following properties are settable:
- **`key`**: (required) a oneline string
- **`args`**:  a list of oneline strings
- **`post-blank`**: a non-negative integer

```el
(->> (om-build-macro "economics")
     (om-to-string))
 ;; => "{{{economics}}}"

(->> (om-build-macro "economics" :args '("s=d"))
     (om-to-string))
 ;; => "{{{economics(s=d)}}}"

```

#### om-build-statistics-cookie `(value &key post-blank)`

Build a statistics-cookie object node.

The following properties are settable:
- **`value`**: (required) a list of non-neg integers like `(perc)` or `(num den)` which make [`num`/`den`] and [`perc`%] respectively
- **`post-blank`**: a non-negative integer

```el
(->> (om-build-statistics-cookie '(nil))
     (om-to-string))
 ;; => "[%]"

(->> (om-build-statistics-cookie '(nil nil))
     (om-to-string))
 ;; => "[/]"

(->> (om-build-statistics-cookie '(50))
     (om-to-string))
 ;; => "[50%]"

(->> (om-build-statistics-cookie '(1 3))
     (om-to-string))
 ;; => "[1/3]"

```

#### om-build-target `(value &key post-blank)`

Build a target object node.

The following properties are settable:
- **`value`**: (required) a oneline string
- **`post-blank`**: a non-negative integer

```el
(->> (om-build-target "text")
     (om-to-string))
 ;; => "<<text>>"

```

#### om-build-timestamp `(type year-start month-start day-start year-end month-end day-end &key hour-start minute-start hour-end minute-end repeater-type repeater-unit repeater-value warning-type warning-unit warning-value post-blank)`

Build a timestamp object node.

The following properties are settable:
- **`type`**: (required) a symbol from `inactive`, `active`, `inactive-ranged`, or `active-ranged`
- **`year-start`**: (required) a positive integer
- **`month-start`**: (required) a positive integer
- **`day-start`**: (required) a positive integer
- **`year-end`**: (required) a positive integer
- **`month-end`**: (required) a positive integer
- **`day-end`**: (required) a positive integer
- **`hour-start`**:  a non-negative integer or nil
- **`minute-start`**:  a non-negative integer or nil
- **`hour-end`**:  a non-negative integer or nil
- **`minute-end`**:  a non-negative integer or nil
- **`repeater-type`**:  nil or a symbol from `catch-up`, `restart`, or `cumulate`
- **`repeater-unit`**:  nil or a symbol from `year` `month` `week` `day`, or `hour`
- **`repeater-value`**:  a positive integer or nil
- **`warning-type`**:  nil or a symbol from `all` or `first`
- **`warning-unit`**:  nil or a symbol from `year` `month` `week` `day`, or `hour`
- **`warning-value`**:  a positive integer or nil
- **`post-blank`**: a non-negative integer

```el
(->> (om-build-timestamp 'inactive
			 2019 1 15 2019 1 15)
     (om-to-string))
 ;; => "[2019-01-15 Tue]"

(->> (om-build-timestamp 'active-range
			 2019 1 15 2019 1 16)
     (om-to-string))
 ;; => "<2019-01-15 Tue>--<2019-01-16 Wed>"

(->> (om-build-timestamp 'inactive
			 2019 1 15 2019 1 15 :warning-type 'all
			 :warning-unit 'day
			 :warning-value 1)
     (om-to-string))
 ;; => "[2019-01-15 Tue -1d]"

```

#### om-build-verbatim `(value &key post-blank)`

Build a verbatim object node.

The following properties are settable:
- **`value`**: (required) a string
- **`post-blank`**: a non-negative integer

```el
(->> (om-build-verbatim "text")
     (om-to-string))
 ;; => "=text="

```


### Branch Object Nodes

#### om-build-bold `(&key post-blank &rest object-nodes)`

Build a bold object node with **`object-nodes`** as children.

The following properties are settable:

- **`post-blank`**: a non-negative integer

```el
(->> (om-build-bold "text")
     (om-to-string))
 ;; => "*text*"

```

#### om-build-footnote-reference `(&key label post-blank &rest object-nodes)`

Build a footnote-reference object node with **`object-nodes`** as children.

The following properties are settable:
- **`label`**:  a oneline string or nil
- **`post-blank`**: a non-negative integer

```el
(->> (om-build-footnote-reference)
     (om-to-string))
 ;; => "[fn:]"

(->> (om-build-footnote-reference :label "label")
     (om-to-string))
 ;; => "[fn:label]"

(->> (om-build-footnote-reference :label "label" "content")
     (om-to-string))
 ;; => "[fn:label:content]"

```

#### om-build-italic `(&key post-blank &rest object-nodes)`

Build an italic object node with **`object-nodes`** as children.

The following properties are settable:

- **`post-blank`**: a non-negative integer

```el
(->> (om-build-italic "text")
     (om-to-string))
 ;; => "/text/"

```

#### om-build-link `(path &key format (type "fuzzy") post-blank &rest object-nodes)`

Build a link object node with **`object-nodes`** as children.

The following properties are settable:
- **`path`**: (required) a oneline string
- **`format`**:  the symbol `plain`, `bracket` or `angle`
- **`type`**: (default `"fuzzy"`) a oneline string from `org-link-types` or `"coderef"`, `"custom-id"`, `"file"`, `"id"`, `"radio"`, or `"fuzzy"`
- **`post-blank`**: a non-negative integer

```el
(->> (om-build-link "target")
     (om-to-string))
 ;; => "[[target]]"

(->> (om-build-link "target" :type "file")
     (om-to-string))
 ;; => "[[file:target]]"

(->> (om-build-link "target" "desc")
     (om-to-string))
 ;; => "[[target][desc]]"

```

#### om-build-radio-target `(&key post-blank &rest object-nodes)`

Build a radio-target object node with **`object-nodes`** as children.

The following properties are settable:

- **`post-blank`**: a non-negative integer

```el
(->> (om-build-radio-target "text")
     (om-to-string))
 ;; => "<<<text>>>"

```

#### om-build-strike-through `(&key post-blank &rest object-nodes)`

Build a strike-through object node with **`object-nodes`** as children.

The following properties are settable:

- **`post-blank`**: a non-negative integer

```el
(->> (om-build-strike-through "text")
     (om-to-string))
 ;; => "+text+"

```

#### om-build-superscript `(&key use-brackets-p post-blank &rest object-nodes)`

Build a superscript object node with **`object-nodes`** as children.

The following properties are settable:
- **`use-brackets-p`**:  nil or t
- **`post-blank`**: a non-negative integer

```el
(->> (om-build-superscript "text")
     (om-to-string))
 ;; => "^text"

```

#### om-build-subscript `(&key use-brackets-p post-blank &rest object-nodes)`

Build a subscript object node with **`object-nodes`** as children.

The following properties are settable:
- **`use-brackets-p`**:  nil or t
- **`post-blank`**: a non-negative integer

```el
(->> (om-build-subscript "text")
     (om-to-string))
 ;; => "_text"

```

#### om-build-table-cell `(&key post-blank &rest object-nodes)`

Build a table-cell object node with **`object-nodes`** as children.

The following properties are settable:

- **`post-blank`**: a non-negative integer

```el
(->> (om-build-table-cell "text")
     (om-to-string))
 ;; => " text |"

```

#### om-build-underline `(&key post-blank &rest object-nodes)`

Build an underline object node with **`object-nodes`** as children.

The following properties are settable:

- **`post-blank`**: a non-negative integer

```el
(->> (om-build-underline "text")
     (om-to-string))
 ;; => "_text_"

```


### Leaf Element Nodes

#### om-build-babel-call `(call &key inside-header arguments end-header post-blank)`

Build a babel-call element node.

The following properties are settable:
- **`call`**: (required) a oneline string
- **`inside-header`**:  a plist
- **`arguments`**:  a list of oneline strings
- **`end-header`**:  a plist
- **`post-blank`**: a non-negative integer

```el
(->> (om-build-babel-call "name")
     (om-to-trimmed-string))
 ;; => "#+CALL: name()"

(->> (om-build-babel-call "name" :arguments '("arg=x"))
     (om-to-trimmed-string))
 ;; => "#+CALL: name(arg=x)"

(->> (om-build-babel-call "name" :inside-header '(:key val))
     (om-to-trimmed-string))
 ;; => "#+CALL: name[:key val]()"

(->> (om-build-babel-call "name" :end-header '(:key val))
     (om-to-trimmed-string))
 ;; => "#+CALL: name() :key val"

```

#### om-build-clock `(value &key post-blank)`

Build a clock element node.

The following properties are settable:
- **`value`**: (required) an unranged, inactive timestamp node with no warning or repeater
- **`post-blank`**: a non-negative integer

```el
(->> (om-build-clock (om-build-timestamp! '(2019 1 1 0 0)))
     (om-to-trimmed-string))
 ;; => "CLOCK: [2019-01-01 Tue 00:00]"

(->> (om-build-clock (om-build-timestamp! '(2019 1 1 0 0)
					  :end '(2019 1 1 1 0)))
     (om-to-trimmed-string))
 ;; => "CLOCK: [2019-01-01 Tue 00:00-01:00] =>  1:00"

```

#### om-build-comment `(value &key post-blank)`

Build a comment element node.

The following properties are settable:
- **`value`**: (required) a string
- **`post-blank`**: a non-negative integer

```el
(->> (om-build-comment "text")
     (om-to-trimmed-string))
 ;; => "# text"

(->> (om-build-comment "text\nless")
     (om-to-trimmed-string))
 ;; => "# text
 ;      # less"

```

#### om-build-comment-block `(&key (value "") post-blank)`

Build a comment-block element node.

The following properties are settable:
- **`value`**: (default `""`) a string
- **`post-blank`**: a non-negative integer

```el
(->> (om-build-comment-block)
     (om-to-trimmed-string))
 ;; => "#+BEGIN_COMMENT
 ;      #+END_COMMENT"

(->> (om-build-comment-block :value "text")
     (om-to-trimmed-string))
 ;; => "#+BEGIN_COMMENT
 ;      text
 ;      #+END_COMMENT"

```

#### om-build-diary-sexp `(&key value post-blank)`

Build a diary-sexp element node.

The following properties are settable:
- **`value`**:  a list form or nil
- **`post-blank`**: a non-negative integer

```el
(->> (om-build-diary-sexp)
     (om-to-trimmed-string))
 ;; => "%%()"

(->> (om-build-diary-sexp :value '(text))
     (om-to-trimmed-string))
 ;; => "%%(text)"

```

#### om-build-example-block `(&key preserve-indent switches (value "") post-blank)`

Build an example-block element node.

The following properties are settable:
- **`preserve-indent`**:  nil or t
- **`switches`**:  a list of oneline strings
- **`value`**: (default `""`) a string
- **`post-blank`**: a non-negative integer

```el
(->> (om-build-example-block)
     (om-to-trimmed-string))
 ;; => "#+BEGIN_EXAMPLE
 ;      #+END_EXAMPLE"

(->> (om-build-example-block :value "text")
     (om-to-trimmed-string))
 ;; => "#+BEGIN_EXAMPLE
 ;      text
 ;      #+END_EXAMPLE"

(->> (om-build-example-block :value "text" :switches '("switches"))
     (om-to-trimmed-string))
 ;; => "#+BEGIN_EXAMPLE switches
 ;      text
 ;      #+END_EXAMPLE"

```

#### om-build-export-block `(type value &key post-blank)`

Build an export-block element node.

The following properties are settable:
- **`type`**: (required) a oneline string
- **`value`**: (required) a string
- **`post-blank`**: a non-negative integer

```el
(->> (om-build-export-block "type" "value\n")
     (om-to-trimmed-string))
 ;; => "#+BEGIN_EXPORT type
 ;      value
 ;      #+END_EXPORT"

```

#### om-build-fixed-width `(value &key post-blank)`

Build a fixed-width element node.

The following properties are settable:
- **`value`**: (required) a oneline string
- **`post-blank`**: a non-negative integer

```el
(->> (om-build-fixed-width "text")
     (om-to-trimmed-string))
 ;; => ": text"

```

#### om-build-horizontal-rule `(&key post-blank)`

Build a horizontal-rule element node.

The following properties are settable:

- **`post-blank`**: a non-negative integer

```el
(->> (om-build-horizontal-rule)
     (om-to-trimmed-string))
 ;; => "-----"

```

#### om-build-keyword `(key value &key post-blank)`

Build a keyword element node.

The following properties are settable:
- **`key`**: (required) a oneline string
- **`value`**: (required) a oneline string
- **`post-blank`**: a non-negative integer

```el
(->> (om-build-keyword "FILETAGS" "tmsu")
     (om-to-trimmed-string))
 ;; => "#+FILETAGS: tmsu"

```

#### om-build-latex-environment `(value &key post-blank)`

Build a latex-environment element node.

The following properties are settable:
- **`value`**: (required) a list of strings like `(env body)` or `(env)`
- **`post-blank`**: a non-negative integer

```el
(->> (om-build-latex-environment '("env" "text"))
     (om-to-trimmed-string))
 ;; => "\\begin{env}
 ;      text
 ;      \\end{env}"

```

#### om-build-node-property `(key value &key post-blank)`

Build a node-property element node.

The following properties are settable:
- **`key`**: (required) a oneline string
- **`value`**: (required) a oneline string
- **`post-blank`**: a non-negative integer

```el
(->> (om-build-node-property "key" "val")
     (om-to-trimmed-string))
 ;; => ":key:      val"

```

#### om-build-planning `(&key closed deadline scheduled post-blank)`

Build a planning element node.

The following properties are settable:
- **`closed`**:  a zero-range, active timestamp node
- **`deadline`**:  a zero-range, active timestamp node
- **`scheduled`**:  a zero-range, active timestamp node
- **`post-blank`**: a non-negative integer

```el
(->> (om-build-planning :closed (om-build-timestamp! '(2019 1 1)
						     :active t))
     (om-to-trimmed-string))
 ;; => "CLOSED: <2019-01-01 Tue>"

(->> (om-build-planning :scheduled (om-build-timestamp! '(2019 1 1)
							:active t))
     (om-to-trimmed-string))
 ;; => "SCHEDULED: <2019-01-01 Tue>"

(->> (om-build-planning :deadline (om-build-timestamp! '(2019 1 1)
						       :active t))
     (om-to-trimmed-string))
 ;; => "DEADLINE: <2019-01-01 Tue>"

```

#### om-build-src-block `(&key (value "") language parameters preserve-indent switches post-blank)`

Build a src-block element node.

The following properties are settable:
- **`value`**: (default `""`) a string
- **`language`**:  a string or nil
- **`parameters`**:  a plist
- **`preserve-indent`**:  nil or t
- **`switches`**:  a list of oneline strings
- **`post-blank`**: a non-negative integer

```el
(->> (om-build-src-block)
     (om-to-trimmed-string))
 ;; => "#+BEGIN_SRC
 ;      #+END_SRC"

(->> (om-build-src-block :value "body")
     (om-to-trimmed-string))
 ;; => "#+BEGIN_SRC
 ;        body
 ;      #+END_SRC"

(->> (om-build-src-block :value "body" :language "emacs-lisp")
     (om-to-trimmed-string))
 ;; => "#+BEGIN_SRC emacs-lisp
 ;        body
 ;      #+END_SRC"

(->> (om-build-src-block :value "body" :switches '("-n 20" "-r"))
     (om-to-trimmed-string))
 ;; => "#+BEGIN_SRC -n 20 -r
 ;        body
 ;      #+END_SRC"

(->> (om-build-src-block :value "body" :parameters '(:key val))
     (om-to-trimmed-string))
 ;; => "#+BEGIN_SRC :key val
 ;        body
 ;      #+END_SRC"

```


### Branch Element Nodes with Child Object Nodes

#### om-build-paragraph `(&key post-blank &rest object-nodes)`

Build a paragraph element node with **`object-nodes`** as children.

The following properties are settable:

- **`post-blank`**: a non-negative integer

```el
(->> (om-build-paragraph "text")
     (om-to-trimmed-string))
 ;; => "text"

```

#### om-build-table-row `(&key post-blank &rest object-nodes)`

Build a table-row element node with **`object-nodes`** as children.

The following properties are settable:

- **`post-blank`**: a non-negative integer

```el
(->> (om-build-table-cell "a")
     (om-build-table-row)
     (om-to-trimmed-string))
 ;; => "| a |"

```

#### om-build-verse-block `(&key post-blank &rest object-nodes)`

Build a verse-block element node with **`object-nodes`** as children.

The following properties are settable:

- **`post-blank`**: a non-negative integer

```el
(->> (om-build-verse-block "text\n")
     (om-to-trimmed-string))
 ;; => "#+BEGIN_VERSE
 ;      text
 ;      #+END_VERSE"

```


### Branch Element Nodes with Child Element Nodes

#### om-build-center-block `(&key post-blank &rest element-nodes)`

Build a center-block element node with **`element-nodes`** as children.

The following properties are settable:

- **`post-blank`**: a non-negative integer

```el
(->> (om-build-center-block)
     (om-to-trimmed-string))
 ;; => "#+BEGIN_CENTER
 ;      #+END_CENTER"

(->> (om-build-paragraph "text")
     (om-build-center-block)
     (om-to-trimmed-string))
 ;; => "#+BEGIN_CENTER
 ;      text
 ;      #+END_CENTER"

```

#### om-build-drawer `(drawer-name &key post-blank &rest element-nodes)`

Build a drawer element node with **`element-nodes`** as children.

The following properties are settable:
- **`drawer-name`**: (required) a oneline string
- **`post-blank`**: a non-negative integer

```el
(->> (om-build-drawer "NAME")
     (om-to-trimmed-string))
 ;; => ":NAME:
 ;      :END:"

(->> (om-build-paragraph "text")
     (om-build-drawer "NAME")
     (om-to-trimmed-string))
 ;; => ":NAME:
 ;      text
 ;      :END:"

```

#### om-build-dynamic-block `(block-name &key arguments post-blank &rest element-nodes)`

Build a dynamic-block element node with **`element-nodes`** as children.

The following properties are settable:
- **`block-name`**: (required) a oneline string
- **`arguments`**:  a plist
- **`post-blank`**: a non-negative integer

```el
(->> (om-build-dynamic-block "empty")
     (om-to-trimmed-string))
 ;; => "#+BEGIN: empty
 ;      #+END:"

(->> (om-build-comment "I'm in here")
     (om-build-dynamic-block "notempty")
     (om-to-trimmed-string))
 ;; => "#+BEGIN: notempty
 ;      # I'm in here
 ;      #+END:"

```

#### om-build-footnote-definition `(label &key post-blank &rest element-nodes)`

Build a footnote-definition element node with **`element-nodes`** as children.

The following properties are settable:
- **`label`**: (required) a oneline string
- **`post-blank`**: a non-negative integer

```el
(->> (om-build-paragraph "footnote contents")
     (om-build-footnote-definition "label")
     (om-to-trimmed-string))
 ;; => "[fn:label] footnote contents"

```

#### om-build-headline `(&key archivedp commentedp footnote-section-p (level 1) (pre-blank 0) priority tags title todo-keyword post-blank &rest element-nodes)`

Build a headline element node with **`element-nodes`** as children.

The following properties are settable:
- **`archivedp`**:  nil or t
- **`commentedp`**:  nil or t
- **`footnote-section-p`**:  nil or t
- **`level`**:  a positive integer
- **`pre-blank`**:  a non-negative integer
- **`priority`**:  an integer between (inclusive) `org-highest-priority` and `org-lowest-priority`
- **`tags`**:  a string list
- **`title`**:  a secondary string
- **`todo-keyword`**:  a oneline string or nil
- **`post-blank`**: a non-negative integer

```el
(->> (om-build-headline)
     (om-to-trimmed-string))
 ;; => "*"

(->> (om-build-headline :level 2 :title '("dummy")
			 :tags '("tmsu"))
     (om-to-trimmed-string))
 ;; => "** dummy            :tmsu:"

(->> (om-build-headline :todo-keyword "TODO" :archivedp t :commentedp t :priority 65)
     (om-to-trimmed-string))
 ;; => "* TODO COMMENT [#A]  :ARCHIVE:"

```

#### om-build-item `(&key (bullet '-) checkbox counter tag post-blank &rest element-nodes)`

Build an item element node with **`element-nodes`** as children.

The following properties are settable:
- **`bullet`**: (default `-`) a positive integer (ordered) or the symbol `-` (unordered)
- **`checkbox`**:  nil or the symbols `on`, `off`, or `trans`
- **`counter`**:  a positive integer or nil
- **`tag`**:  a secondary string
- **`post-blank`**: a non-negative integer

```el
(->> (om-build-paragraph "item contents")
     (om-build-item)
     (om-to-trimmed-string))
 ;; => "- item contents"

(->> (om-build-paragraph "item contents")
     (om-build-item :bullet 1)
     (om-to-trimmed-string))
 ;; => "1. item contents"

(->> (om-build-paragraph "item contents")
     (om-build-item :checkbox 'on)
     (om-to-trimmed-string))
 ;; => "- [X] item contents"

(->> (om-build-paragraph "item contents")
     (om-build-item :tag '("tmsu"))
     (om-to-trimmed-string))
 ;; => "- tmsu :: item contents"

(->> (om-build-paragraph "item contents")
     (om-build-item :counter 10)
     (om-to-trimmed-string))
 ;; => "- [@10] item contents"

```

#### om-build-plain-list `(&key post-blank &rest element-nodes)`

Build a plain-list element node with **`element-nodes`** as children.

The following properties are settable:

- **`post-blank`**: a non-negative integer

```el
(->> (om-build-paragraph "item contents")
     (om-build-item)
     (om-build-plain-list)
     (om-to-trimmed-string))
 ;; => "- item contents"

```

#### om-build-property-drawer `(&key post-blank &rest element-nodes)`

Build a property-drawer element node with **`element-nodes`** as children.

The following properties are settable:

- **`post-blank`**: a non-negative integer

```el
(->> (om-build-property-drawer)
     (om-to-trimmed-string))
 ;; => ":PROPERTIES:
 ;      :END:"

(->> (om-build-node-property "key" "val")
     (om-build-property-drawer)
     (om-to-trimmed-string))
 ;; => ":PROPERTIES:
 ;      :key:      val
 ;      :END:"

```

#### om-build-quote-block `(&key post-blank &rest element-nodes)`

Build a quote-block element node with **`element-nodes`** as children.

The following properties are settable:

- **`post-blank`**: a non-negative integer

```el
(->> (om-build-quote-block)
     (om-to-trimmed-string))
 ;; => "#+BEGIN_QUOTE
 ;      #+END_QUOTE"

(->> (om-build-paragraph "quoted stuff")
     (om-build-quote-block)
     (om-to-trimmed-string))
 ;; => "#+BEGIN_QUOTE
 ;      quoted stuff
 ;      #+END_QUOTE"

```

#### om-build-section `(&key post-blank &rest element-nodes)`

Build a section element node with **`element-nodes`** as children.

The following properties are settable:

- **`post-blank`**: a non-negative integer

```el
(->> (om-build-paragraph "text")
     (om-build-section)
     (om-to-trimmed-string))
 ;; => "text"

```

#### om-build-special-block `(type &key post-blank &rest element-nodes)`

Build a special-block element node with **`element-nodes`** as children.

The following properties are settable:
- **`type`**: (required) a oneline string
- **`post-blank`**: a non-negative integer

```el
(->> (om-build-special-block "monad")
     (om-to-trimmed-string))
 ;; => "#+BEGIN_monad
 ;      #+END_monad"

(->> (om-build-comment "Launch missiles")
     (om-build-special-block "monad")
     (om-to-trimmed-string))
 ;; => "#+BEGIN_monad
 ;      # Launch missiles
 ;      #+END_monad"

```

#### om-build-table `(&key tblfm post-blank &rest element-nodes)`

Build a table element node with **`element-nodes`** as children.

The following properties are settable:
- **`tblfm`**:  a list of oneline strings
- **`post-blank`**: a non-negative integer

```el
(->> (om-build-table-cell "cell")
     (om-build-table-row)
     (om-build-table)
     (om-to-trimmed-string))
 ;; => "| cell |"

```


### Miscellaneous Builders

#### om-build-secondary-string! `(string)`

Return a secondary string (list of object nodes) from **`string`**.
**`string`** is any string that contains a textual representation of
object nodes. If the string does not represent a list of object nodes,
throw an error.

```el
(->> (om-build-secondary-string! "I'm plain")
     (-map (function om-get-type)))
 ;; => '(plain-text)

(->> (om-build-secondary-string! "I'm *not* plain")
     (-map (function om-get-type)))
 ;; => '(plain-text bold plain-text)

(->> (om-build-secondary-string! "* I'm not an object")
     (-map (function om-get-type)))
Error

```

#### om-build-table-row-hline `(&key post-blank)`

Return a new rule-typed table-row node.
Optionally set **`post-blank`** (a positive integer).

```el
(->> (om-build-table (om-build-table-row (om-build-table-cell "text"))
		     (om-build-table-row-hline))
     (om-to-trimmed-string))
 ;; => "| text |
 ;      |------|"

```

#### om-build-timestamp-diary `(form &key post-blank)`

Return a new diary-sexp timestamp node from **`form`**.
Optionally set **`post-blank`** (a positive integer).

```el
(->> (om-build-timestamp-diary '(diary-float t 4 2))
     (om-to-string))
 ;; => "<%%(diary-float t 4 2)>"

```


### Shorthand Builders


Build nodes with more convenient/shorter syntax.

#### om-build-timestamp! `(start &key end active repeater warning post-blank)`

Return a new timestamp node.

**`start`** specifies the start time and is a list of integers in one of
the following forms:
- `(year month day)`: short form
- `(year month day nil nil)`: short form
- `(year month day hour minute)`: long form

**`end`** (if supplied) will add the ending time, and follows the same
formatting rules as **`start`**.

**`active`** is a boolean where t signifies the type is `active`, else
`inactive` (the range suffix will be added if an end time is
supplied).

**`repeater`** and **`warning`** are lists formatted as `(type value unit)` where
the three members correspond to the :repeater/warning-type, -value,
and -unit properties in [`om-build-timestamp`](#om-build-timestamp-type-year-start-month-start-day-start-year-end-month-end-day-end-key-hour-start-minute-start-hour-end-minute-end-repeater-type-repeater-unit-repeater-value-warning-type-warning-unit-warning-value-post-blank).

Building a diary sexp timestamp is not possible with this function.

```el
(->> (om-build-timestamp! '(2019 1 1))
     (om-to-string))
 ;; => "[2019-01-01 Tue]"

(->> (om-build-timestamp! '(2019 1 1 12 0)
			  :active t :warning '(all 1 day)
			  :repeater '(cumulate 1 month))
     (om-to-string))
 ;; => "<2019-01-01 Tue 12:00 +1m -1d>"

(->> (om-build-timestamp! '(2019 1 1)
			  :end '(2019 1 2))
     (om-to-string))
 ;; => "[2019-01-01 Tue]--[2019-01-02 Wed]"

```

#### om-build-clock! `(start &key end post-blank)`

Return a new clock node.

**`start`** and **`end`** follow the same rules as their respective arguments in
[`om-build-timestamp!`](#om-build-timestamp-start-key-end-active-repeater-warning-post-blank).

```el
(->> (om-build-clock! '(2019 1 1))
     (om-to-trimmed-string))
 ;; => "CLOCK: [2019-01-01 Tue]"

(->> (om-build-clock! '(2019 1 1 12 0))
     (om-to-trimmed-string))
 ;; => "CLOCK: [2019-01-01 Tue 12:00]"

(->> (om-build-clock! '(2019 1 1 12 0)
		      :end '(2019 1 1 13 0))
     (om-to-trimmed-string))
 ;; => "CLOCK: [2019-01-01 Tue 12:00-13:00] =>  1:00"

```

#### om-build-planning! `(&key closed deadline scheduled post-blank)`

Return a new planning node.

**`closed`**, **`deadline`**, and **`scheduled`** are lists with the following structure
(brackets denote optional members):

`(year minute day [hour] [min]
 [&warning type value unit]
 [&repeater type value unit])`

In terms of arguments supplied to [`om-build-timestamp!`](#om-build-timestamp-start-key-end-active-repeater-warning-post-blank), the first
five members correspond to the list supplied as `time`, and the `type`,
`value`, and `unit` fields correspond to the lists supplied to `warning` and
`repeater` arguments. The order of warning and repeater does not
matter.

```el
(->> (om-build-planning! :closed '(2019 1 1))
     (om-to-trimmed-string))
 ;; => "CLOSED: <2019-01-01 Tue>"

(->> (om-build-planning! :closed '(2019 1 1)
			  :scheduled '(2018 1 1))
     (om-to-trimmed-string))
 ;; => "SCHEDULED: <2018-01-01 Mon> CLOSED: <2019-01-01 Tue>"

(->> (om-build-planning! :closed '(2019 1 1 &warning all 1 day &repeater cumulate 1 month))
     (om-to-trimmed-string))
 ;; => "CLOSED: <2019-01-01 Tue +1m -1d>"

```

#### om-build-property-drawer! `(&key post-blank &rest keyvals)`

Return a new property-drawer node.

Each member in **`keyvals`** is a list of symbols like `(key val)`, where each
list will generate a node-property node in the property-drawer node
like `":key: val"`.

```el
(->> (om-build-property-drawer! '(key val))
     (om-to-trimmed-string))
 ;; => ":PROPERTIES:
 ;      :key:      val
 ;      :END:"

```

#### om-build-headline! `(&key (level 1) title-text todo-keyword tags pre-blank priority commentedp archivedp post-blank planning statistics-cookie section-children &rest subheadlines)`

Return a new headline node.

**`title-text`** is a oneline string for the title of the headline.

**`planning`** is a list like `(planning-type args ...)` where
`planning-type` is one of `:closed`, `:deadline`, or `:scheduled`, and
`args` are the args supplied to any of the planning types in
[`om-build-planning!`](#om-build-planning-key-closed-deadline-scheduled-post-blank). Up to all three planning types can be used
in the same list like `(:closed args :deadline args :scheduled args)`.

**`statistics-cookie`** is a list following the same format as
[`om-build-statistics-cookie`](#om-build-statistics-cookie-value-key-post-blank).

**`section-children`** is a list of elements that will go in the headline
section.

**`subheadlines`** contains zero or more headlines that will go under the
created headline. The level of all members in **`subheadlines`** will
automatically be adjusted to **`level`** + 1.

All arguments not mentioned here follow the same rules as
[`om-build-headline`](#om-build-headline-key-archivedp-commentedp-footnote-section-p-level-1-pre-blank-0-priority-tags-title-todo-keyword-post-blank-rest-element-nodes)

```el
(->> (om-build-headline! :title-text "really impressive title")
     (om-to-trimmed-string))
 ;; => "* really impressive title"

(->> (om-build-headline! :title-text "really impressive title" :statistics-cookie '(0 9000))
     (om-to-trimmed-string))
 ;; => "* really impressive title [0/9000]"

(->> (om-build-headline! :title-text "really impressive title" :section-children (list (om-build-property-drawer! '(key val))
										       (om-build-paragraph! "section text"))
			  (om-build-headline! :title-text "subhead"))
     (om-to-trimmed-string))
 ;; => "* really impressive title
 ;      :PROPERTIES:
 ;      :key:      val
 ;      :END:
 ;      section text
 ;      ** subhead"

```

#### om-build-item! `(&key post-blank bullet checkbox tag paragraph counter &rest children)`

Return a new item node.

**`tag`** is a string representing the tag (make with
[`om-build-secondary-string!`](#om-build-secondary-string-string)) .

**`paragraph`** is a string that will be the initial text in the item
(made with [`om-build-paragraph!`](#om-build-paragraph-string-key-post-blank)).

**`children`** contains the nodes that will go under this item after
**`paragraph`**.

All other arguments follow the same rules as [`om-build-item`](#om-build-item-key-bullet-quote---checkbox-counter-tag-post-blank-rest-element-nodes).

```el
(->> (om-build-item! :bullet 1 :tag "complicated *tag*" :paragraph "petulant /frenzy/" (om-build-plain-list (om-build-item! :bullet '-
															     :paragraph "below")))
     (om-to-trimmed-string))
 ;; => "1. complicated *tag* :: petulant /frenzy/
 ;         - below"

```

#### om-build-paragraph! `(string &key post-blank)`

Return a new paragraph node from **`string`**.

**`string`** is the text to be parsed into a paragraph and must contain
valid textual representations of object nodes.

```el
(->> (om-build-paragraph! "stuff /with/ *formatting*" :post-blank 2)
     (om-to-string))
 ;; => "stuff /with/ *formatting*
 ;      
 ;      
 ;      "

(->> (om-build-paragraph! "* stuff /with/ *formatting*")
     (om-to-string))
Error

```

#### om-build-table-cell! `(string)`

Return a new table-cell node.

**`string`** is the text to be contained in the table-cell node. It must
contain valid textual representations of objects that are allowed in
table-cell nodes.

```el
(->> (om-build-table-cell! "rage")
     (om-to-trimmed-string))
 ;; => "rage |"

(->> (om-build-table-cell! "*rage*")
     (om-to-trimmed-string))
 ;; => "*rage* |"

```

#### om-build-table-row! `(row-list)`

Return a new table-row node.

**`row-list`** is a list of strings to be built into table-cell nodes via
[`om-build-table-cell!`](#om-build-table-cell-string) (see that function for restrictions).
Alternatively, **`row-list`** may the symbol `hline` instead of a string to
create a rule-typed table-row.

```el
(->> (om-build-table-row! '("R" "A" "G" "E"))
     (om-to-trimmed-string))
 ;; => "| R | A | G | E |"

(->> (om-build-table-row! 'hline)
     (om-to-trimmed-string))
 ;; => "|-"

```

#### om-build-table! `(&key tblfm post-blank &rest row-lists)`

Return a new table node.

**`row-lists`** is a list of lists where each member list will be converted
to a table-row node via [`om-build-table-row!`](#om-build-table-row-row-list) (see that function for
restrictions).

All other arguments follow the same rules as [`om-build-table`](#om-build-table-key-tblfm-post-blank-rest-element-nodes).

```el
(->> (om-build-table! '("R" "A")
		      '("G" "E"))
     (om-to-trimmed-string))
 ;; => "| R | A |
 ;      | G | E |"

(->> (om-build-table! '("L" "O")
		      'hline
		      '("V" "E"))
     (om-to-trimmed-string))
 ;; => "| L | O |
 ;      |---+---|
 ;      | V | E |"

```


## Type Predicates


Test node types.

#### om-get-type `(node)`

Return the type of **`node`**.

```el
;; Given the following contents:
; *I'm emboldened*

(->> (om-parse-this-object)
     (om-get-type))
 ;; => 'bold

;; Given the following contents:
; * I'm the headliner

(->> (om-parse-this-element)
     (om-get-type))
 ;; => 'headline

;; Given the following contents:
; [2112-12-21 Wed]

(->> (om-parse-this-object)
     (om-get-type))
 ;; => 'timestamp

```

#### om-is-type `(type node)`

Return t if the type of **`node`** is **`type`** (a symbol).

```el
;; Given the following contents:
; *ziltoid*

(->> (om-parse-this-object)
     (om-is-type 'bold))
 ;; => t

(->> (om-parse-this-object)
     (om-is-type 'italic))
 ;; => nil

```

#### om-is-any-type `(types node)`

Return t if the type of **`node`** is in **`types`** (a list of symbols).

```el
;; Given the following contents:
; *ziltoid*

(->> (om-parse-this-object)
     (om-is-any-type '(bold)))
 ;; => t

(->> (om-parse-this-object)
     (om-is-any-type '(bold italic)))
 ;; => t

(->> (om-parse-this-object)
     (om-is-any-type '(italic)))
 ;; => nil

```

#### om-is-element `(node)`

Return t if **`node`** is an element class.

```el
;; Given the following contents:
; *ziltoid*

;; Parsing this text as an element node gives a paragraph node
(->> (om-parse-this-element)
     (om-is-element))
 ;; => t

;; Parsing the same text as an object node gives a bold node
(->> (om-parse-this-object)
     (om-is-element))
 ;; => nil

```

#### om-is-branch-node `(node)`

Return t if **`node`** is a branch node.

```el
;; Given the following contents:
; *ziltoid*

;; Parsing this as an element node gives a paragraph node (a branch node)
(->> (om-parse-this-element)
     (om-is-branch-node))
 ;; => t

;; Parsing this as an object node gives a bold node (also a branch node)
(->> (om-parse-this-object)
     (om-is-branch-node))
 ;; => t

;; Given the following contents:
; ~ziltoid~

;; Parsing this as an object node gives a code node (not a branch node)
(->> (om-parse-this-object)
     (om-is-branch-node))
 ;; => nil

;; Given the following contents:
; # ziltoid

;; Parsing this as an element node gives a comment node (also not a branch node)
(->> (om-parse-this-element)
     (om-is-branch-node))
 ;; => nil

;; Given the following contents:
; * I'm so great

;; Parsing this as an element node gives a headline node (a branch node)
(->> (om-parse-this-element)
     (om-is-branch-node))
 ;; => t

```

#### om-node-may-have-child-objects `(node)`

Return t if **`node`** is a branch node that may have child objects.

```el
;; Given the following contents:
; *ziltoid*

;; Parsing this as an element node gives a paragraph node (can have child object
;; nodes)
(->> (om-parse-this-element)
     (om-node-may-have-child-objects))
 ;; => t

;; Parsing this as an object node gives a bold node (also can have child object
;; nodes)
(->> (om-parse-this-object)
     (om-node-may-have-child-objects))
 ;; => t

;; Given the following contents:
; ~ziltoid~

;; Parsing this as an object node gives a code node (not a branch node)
(->> (om-parse-this-object)
     (om-node-may-have-child-objects))
 ;; => nil

;; Given the following contents:
; # ziltoid

;; Parsing this as an element node gives a comment node (not a branch node)
(->> (om-parse-this-element)
     (om-node-may-have-child-objects))
 ;; => nil

;; Given the following contents:
; * I'm so great

;; Parsing this as an element node gives a headline node (can only have child
;; element nodes)
(->> (om-parse-this-element)
     (om-node-may-have-child-objects))
 ;; => nil

```

#### om-node-may-have-child-elements `(node)`

Return t if **`node`** is a branch node that may have child elements.

Note this implies that **`node`** is also of class element since only
elements may have other elements as children.

```el
;; Given the following contents:
; * I'm so great

;; Parsing this as an element node gives a headline node (can have child element
;; nodes)
(->> (om-parse-this-element)
     (om-node-may-have-child-elements))
 ;; => t

;; Given the following contents:
; *ziltoid*

;; Parsing this as an element node gives a paragraph node (can only have child
;; object nodes)
(->> (om-parse-this-element)
     (om-node-may-have-child-elements))
 ;; => nil

;; Given the following contents:
; # ziltoid

;; Parsing this as an element node gives a comment node (not a branch node)
(->> (om-parse-this-element)
     (om-node-may-have-child-elements))
 ;; => nil

```


## Property Manipulation


Set, get, and map properties of nodes.


### Generic

#### om-contains-point-p `(point node)`

Return t if **`point`** is within the boundaries of **`node`**.

```el
;; Given the following contents:
; *findme*

(->> (om-parse-this-object)
     (om-contains-point-p 2))
 ;; => t

(->> (om-parse-this-object)
     (om-contains-point-p 10))
 ;; => nil

```

#### om-set-property `(prop value node)`

Return **`node`** with **`prop`** set to **`value`**.

See builder functions for a list of properties and their rules for
each type.

```el
;; Given the following contents:
; #+CALL: ktulu()

(->> (om-parse-this-element)
     (om-set-property :call "cthulhu")
     (om-set-property :inside-header '(:cache no))
     (om-set-property :arguments '("x=4"))
     (om-set-property :end-header '(:exports results))
     (om-to-trimmed-string))
 ;; => "#+CALL: cthulhu[:cache no](x=4) :exports results"

;; Given the following contents:
; call_kthulu()

(->> (om-parse-this-object)
     (om-set-property :call "cthulhu")
     (om-set-property :inside-header '(:cache no))
     (om-set-property :arguments '("x=4"))
     (om-set-property :end-header '(:exports results))
     (om-to-trimmed-string))
 ;; => "call_cthulhu[:cache no](x=4)[:exports results]"

;; Given the following contents:
; src_emacs{(print 'yeah-boi)}

(->> (om-parse-this-object)
     (om-set-property :language "python")
     (om-set-property :parameters '(:cache no))
     (om-set-property :value "print \"yeah boi\"")
     (om-to-trimmed-string))
 ;; => "src_python[:cache no]{print \"yeah boi\"}"

;; Given the following contents:
; - thing

(->> (om-parse-this-item)
     (om-set-property :bullet 1)
     (om-set-property :checkbox 'on)
     (om-set-property :counter 2)
     (om-set-property :tag '("tmsu"))
     (om-to-trimmed-string))
 ;; => "1. [@2] [X] tmsu :: thing"

;; Given the following contents:
; * not valuable

;; Throw error when setting a property that doesn't exist
(->> (om-parse-this-headline)
     (om-set-property :value "wtf")
     (om-to-trimmed-string))
Error

;; Throw error when setting to an improper type
(->> (om-parse-this-headline)
     (om-set-property :title 666)
     (om-to-trimmed-string))
Error

```

#### om-set-properties `(plist node)`

Return **`node`** with all properties set to the values according to **`plist`**.

**`plist`** is a list of property-value pairs that corresponds to the
property list in **`node`**.

See builder functions for a list of properties and their rules for
each type.

```el
;; Given the following contents:
; - thing

(->> (om-parse-this-item)
     (om-set-properties (list :bullet 1 :checkbox 'on
			       :counter 2 :tag '("tmsu")))
     (om-to-trimmed-string))
 ;; => "1. [@2] [X] tmsu :: thing"

```

#### om-get-property `(prop node)`

Return the value of **`prop`** of **`node`**.

See builder functions for a list of properties and their rules for
each type.

```el
;; Given the following contents:
; #+CALL: ktulu(x=4) :exports results

(->> (om-parse-this-element)
     (om-get-property :call))
 ;; => "ktulu"

(->> (om-parse-this-element)
     (om-get-property :inside-header))
 ;; => nil

;; Given the following contents:
; * not arguable

;; Throw error when requesting a property that doesn't exist
(->> (om-parse-this-headline)
     (om-get-property :value))
Error

```

#### om-map-property `(prop fun node)`

Return **`node`** with **`fun`** applied to the value of **`prop`**.

**`fun`** is a unary function which takes the current value of **`prop`** and
returns a new value to which **`prop`** will be set.

See builder functions for a list of properties and their rules for
each type.

```el
;; Given the following contents:
; ~learn to~

(->> (om-parse-this-object)
     (om-map-property :value (function s-upcase))
     (om-to-trimmed-string))
 ;; => "~LEARN TO~"

;; Throw error if property doesn't exist
(->> (om-parse-this-object)
     (om-map-property :title (function s-upcase))
     (om-to-trimmed-string))
Error

;; Throw error if function doesn't return proper type
(->> (om-parse-this-object)
     (om-map-property* :value (if it 1 0))
     (om-to-trimmed-string))
Error

```

#### om-map-properties `(plist node)`

Return **`node`** with functions applied to the values of properties.

**`plist`** is a property list where the keys are properties of **`node`** and
its values are unary functions to be mapped to these properties.

See builder functions for a list of properties and their rules for
each type.

```el
;; Given the following contents:
; #+KEY: VAL

(->> (om-parse-this-element)
     (om-map-properties (list :key (-partial (function s-prepend)
					     "OM_")
			       :value (-partial (function s-prepend)
						"OM_")))
     (om-to-trimmed-string))
 ;; => "#+OM_KEY: OM_VAL"

```

#### om-toggle-property `(prop node)`

Return **`node`** with the value of **`prop`** flipped.

This function only applies to properties that are booleans.

The following types and properties are supported:

entity
- :use-brackets-p

example-block
- :preserve-indent

headline
- :archivedp
- :commentedp
- :footnote-section-p

src-block
- :preserve-indent

subscript
- :use-brackets-p

superscript
- :use-brackets-p

```el
;; Given the following contents:
; \pi

(->> (om-parse-this-object)
     (om-toggle-property :use-brackets-p)
     (om-to-trimmed-string))
 ;; => "\\pi{}"

;; Given the following contents:
; - [ ] nope

;; Throw an error when trying to toggle a non-boolean property
(->> (om-parse-this-item)
     (om-toggle-property :checkbox)
     (om-to-trimmed-string))
Error

```

#### om-shift-property `(prop n node)`

Return **`node`** with **`prop`** shifted by **`n`** (an integer).

This only applies the properties that are represented as integers.

The following types and properties are supported:

all elements
- :post-blank

headline
- :level
- :pre-blank
- :priority

item
- :counter

```el
;; Given the following contents:
; * no priorities

;; Do nothing if there is nothing to shift.
(->> (om-parse-this-headline)
     (om-shift-property :priority 1)
     (om-to-trimmed-string))
 ;; => "* no priorities"

;; Given the following contents:
; * [#A] priorities

(->> (om-parse-this-headline)
     (om-shift-property :priority -1)
     (om-to-trimmed-string))
 ;; => "* [#B] priorities"

;; Wrap priority around when crossing the min or max
(->> (om-parse-this-headline)
     (om-shift-property :priority 1)
     (om-to-trimmed-string))
 ;; => "* [#C] priorities"

;; Given the following contents:
; * TODO or not todo

;; Throw error when shifting an unshiftable property
(->> (om-parse-this-headline)
     (om-shift-property :todo-keyword 1)
     (om-to-string))
Error

```

#### om-insert-into-property `(prop index string node)`

Return **`node`** with **`string`** inserted at **`index`** into **`prop`**.

This only applies to properties that are represented as lists of
strings.

The following types and properties are supported:

babel-call
- :arguments

example-block
- :switches

headline
- :tags

inline-babel-call
- :arguments

macro
- :args

src-block
- :switches

table
- :tblfm

```el
;; Given the following contents:
; #+CALL: ktulu(y=1)

(->> (om-parse-this-element)
     (om-insert-into-property :arguments 0 "x=4")
     (om-to-trimmed-string))
 ;; => "#+CALL: ktulu(x=4,y=1)"

;; Do nothing if the string is already in the list
(->> (om-parse-this-element)
     (om-insert-into-property :arguments 0 "y=1")
     (om-to-trimmed-string))
 ;; => "#+CALL: ktulu(y=1)"

;; Throw error when inserting into a property that is not a list of strings
(->> (om-parse-this-element)
     (om-insert-into-property :end-header 0 "html")
     (om-to-trimmed-string))
Error

```

#### om-remove-from-property `(prop string node)`

Return **`node`** with **`string`** removed from **`prop`** if present.

This only applies to properties that are represented as lists of
strings.

See [`om-insert-into-property`](#om-insert-into-property-prop-index-string-node) for a list of supported elements
and properties that may be used with this function.

```el
;; Given the following contents:
; #+CALL: ktulu(y=1)

(->> (om-parse-this-element)
     (om-remove-from-property :arguments "y=1")
     (om-to-trimmed-string))
 ;; => "#+CALL: ktulu()"

;; Do nothing if the string does not exist
(->> (om-parse-this-element)
     (om-remove-from-property :arguments "d=666")
     (om-to-trimmed-string))
 ;; => "#+CALL: ktulu(y=1)"

;; Throw error when removing from property that is not a string list
(->> (om-parse-this-element)
     (om-remove-from-property :end-header ":results")
     (om-to-trimmed-string))
Error

```

#### om-plist-put-property `(prop key value node)`

Return **`node`** with **`value`** corresponding to **`key`** inserted into **`prop`**.

**`key`** is a keyword and **`value`** is a symbol. This only applies to
properties that are represented as plists.

The following types and properties are supported:

babel-call
- :inside-header
- :end-header

dynamic-block
- :arguments

inline-babel-call
- :inside-header
- :end-header

inline-src-block
- :parameters

src-block
- :parameters

```el
;; Given the following contents:
; #+CALL: ktulu[:cache no]()

(->> (om-parse-this-element)
     (om-plist-put-property :end-header :results 'html)
     (om-to-trimmed-string))
 ;; => "#+CALL: ktulu[:cache no]() :results html"

;; Change the value of key if it already is present
(->> (om-parse-this-element)
     (om-plist-put-property :inside-header :cache 'yes)
     (om-to-trimmed-string))
 ;; => "#+CALL: ktulu[:cache yes]()"

;; Do nothing if the key and value already exist
(->> (om-parse-this-element)
     (om-plist-put-property :inside-header :cache 'no)
     (om-to-trimmed-string))
 ;; => "#+CALL: ktulu[:cache no]()"

;; Throw error if setting property that isn't a plist
(->> (om-parse-this-element)
     (om-plist-put-property :arguments :cache 'no)
     (om-to-trimmed-string))
Error

```

#### om-plist-remove-property `(prop key node)`

Return **`node`** with **`key`** and its corresponding value removed from **`prop`**.

**`key`** is a keyword. This only applies to properties that are
represented as plists.

See [`om-plist-put-property`](#om-plist-put-property-prop-key-value-node) for a list of supported elements
and properties that may be used with this function.

```el
;; Given the following contents:
; #+CALL: ktulu() :results html

(->> (om-parse-this-element)
     (om-plist-remove-property :end-header :results)
     (om-to-trimmed-string))
 ;; => "#+CALL: ktulu()"

;; Do nothing if the key is not present
(->> (om-parse-this-element)
     (om-plist-remove-property :inside-header :cache)
     (om-to-trimmed-string))
 ;; => "#+CALL: ktulu() :results html"

;; Throw error if trying to remove key from non-plist property
(->> (om-parse-this-element)
     (om-plist-remove-property :arguments :cache)
     (om-to-trimmed-string))
Error

```


### Clock

#### om-clock-is-running `(clock)`

Return t if **`clock`** element is running (eg is open).

```el
;; Given the following contents:
; CLOCK: [2019-01-01 Tue 00:00]

(->> (om-parse-this-element)
     (om-clock-is-running))
 ;; => t

;; Given the following contents:
; CLOCK: [2019-01-01 Tue 00:00]--[2019-01-02 Wed 00:00] => 24:00

(->> (om-parse-this-element)
     (om-clock-is-running))
 ;; => nil

```


### Entity

#### om-entity-get-replacement `(key entity)`

Return replacement string or symbol for **`entity`** node.

**`key`** is one of:
- `:latex` (the entity's latex representation)
- `:latex-math-p` (t if the latex representation requires math mode,
    nil otherwise)
- `:html` (the entity's html representation)
- `:ascii` (the entity's ascii representation)
- `:latin1` (the entity's Latin1 representation)
- `:utf-8` (the entity's utf8 representation)

Any other keys will trigger an error.

```el
;; Given the following contents:
; \pi{}

(->> (om-parse-this-object)
     (om-entity-get-replacement :latex))
 ;; => "\\pi"

(->> (om-parse-this-object)
     (om-entity-get-replacement :latex-math-p))
 ;; => t

(->> (om-parse-this-object)
     (om-entity-get-replacement :html))
 ;; => "&pi;"

(->> (om-parse-this-object)
     (om-entity-get-replacement :ascii))
 ;; => "pi"

(->> (om-parse-this-object)
     (om-entity-get-replacement :latin1))
 ;; => "pi"

(->> (om-parse-this-object)
     (om-entity-get-replacement :utf-8))
 ;; => "π"

```


### Headline

#### om-headline-set-title! `(title-text stats-cookie-value headline)`

Return **`headline`** node with title set with **`title-text`** and **`stats-cookie-value`**.

**`title-text`** is a string to be parsed into object nodes for the title
via [`om-build-secondary-string!`](#om-build-secondary-string-string) (see that function for restrictions)
and **`stats-cookie-value`** is a list described in
[`om-build-statistics-cookie`](#om-build-statistics-cookie-value-key-post-blank).

```el
;; Given the following contents:
; * really impressive title

(->> (om-parse-this-headline)
     (om-headline-set-title! "really *impressive* title" '(2 3))
     (om-to-trimmed-string))
 ;; => "* really *impressive* title [2/3]"

```

#### om-headline-is-done `(headline)`

Return t if **`headline`** node has a done todo-keyword.

```el
;; Given the following contents:
; * TODO darn

(->> (om-parse-this-headline)
     (om-headline-is-done))
 ;; => nil

;; Given the following contents:
; * DONE yay

(->> (om-parse-this-headline)
     (om-headline-is-done))
 ;; => t

```

#### om-headline-has-tag `(tag headline)`

Return t if **`headline`** node is tagged with **`tag`**.

```el
;; Given the following contents:
; * dummy

(->> (om-parse-this-headline)
     (om-headline-has-tag "tmsu"))
 ;; => nil

;; Given the following contents:
; * dummy                  :tmsu:

(->> (om-parse-this-headline)
     (om-headline-has-tag "tmsu"))
 ;; => t

```

#### om-headline-get-statistics-cookie `(headline)`

Return the statistics cookie node from **`headline`** if it exists.

```el
;; Given the following contents:
; * statistically significant [10/10]

(->> (om-parse-this-headline)
     (om-headline-get-statistics-cookie)
     (om-to-string))
 ;; => "[10/10]"

;; Given the following contents:
; * not statistically significant

(->> (om-parse-this-headline)
     (om-headline-get-statistics-cookie))
 ;; => nil

```


### Item

#### om-item-toggle-checkbox `(item)`

Return **`item`** node with its checkbox state flipped.
This only affects item nodes with checkboxes in the `on` or `off`
states; return **`item`** node unchanged if the checkbox property is `trans`
or nil.

```el
;; Given the following contents:
; - [ ] one

(->> (om-parse-this-item)
     (om-item-toggle-checkbox)
     (om-to-trimmed-string))
 ;; => "- [X] one"

;; Given the following contents:
; - [-] one

;; Ignore trans state checkboxes
(->> (om-parse-this-item)
     (om-item-toggle-checkbox)
     (om-to-trimmed-string))
 ;; => "- [-] one"

;; Given the following contents:
; - one

;; Do nothing if there is no checkbox
(->> (om-parse-this-item)
     (om-item-toggle-checkbox)
     (om-to-trimmed-string))
 ;; => "- one"

```


### Planning

#### om-planning-set-timestamp! `(prop planning-list planning)`

Return **`planning`** node with **`prop`** set to **`planning-list`**.

**`prop`** is one of `:closed`, `:deadline`, or `:scheduled`. **`planning-list`**
is the same as that described in [`om-build-planning!`](#om-build-planning-key-closed-deadline-scheduled-post-blank).

```el
;; Given the following contents:
; * dummy
; CLOSED: <2019-01-01 Tue>

;; Change an existing timestamp in planning
(->> (om-parse-this-headline)
     (om-headline-get-planning)
     (om-planning-set-timestamp! :closed '(2019 1 2 &warning all 1 day &repeater cumulate 2 month))
     (om-to-trimmed-string))
 ;; => "CLOSED: <2019-01-02 Wed +2m -1d>"

;; Add a new timestamp and remove another
(->> (om-parse-this-headline)
     (om-headline-get-planning)
     (om-planning-set-timestamp! :deadline '(2112 1 1))
     (om-planning-set-timestamp! :closed nil)
     (om-to-trimmed-string))
 ;; => "DEADLINE: <2112-01-01 Fri>"

```


### Statistics Cookie

#### om-statistics-cookie-is-complete `(statistics-cookie)`

Return t is **`statistics-cookie`** node is complete.

```el
;; Given the following contents:
; * statistically significant [10/10]

(->> (om-parse-this-headline)
     (om-headline-get-statistics-cookie)
     (om-statistics-cookie-is-complete))
 ;; => t

;; Given the following contents:
; * statistically significant [1/10]

(->> (om-parse-this-headline)
     (om-headline-get-statistics-cookie)
     (om-statistics-cookie-is-complete))
 ;; => nil

;; Given the following contents:
; * statistically significant [100%]

(->> (om-parse-this-headline)
     (om-headline-get-statistics-cookie)
     (om-statistics-cookie-is-complete))
 ;; => t

;; Given the following contents:
; * statistically significant [33%]

(->> (om-parse-this-headline)
     (om-headline-get-statistics-cookie)
     (om-statistics-cookie-is-complete))
 ;; => nil

```


### Timestamp (Auxiliary)


Functions to work with timestamp data

#### om-time-to-unixtime `(time)`

Return the unix time (integer seconds) of time list **`time`**.
The returned value is dependent on the time zone of the operating
system.

```el
no examples :(
```

#### om-unixtime-to-time-long `(unixtime)`

Return the long time list of **`unixtime`**.
The list will be formatted like `(year month day hour min)`.

```el
no examples :(
```

#### om-unixtime-to-time-short `(unixtime)`

Return the short time list of **`unixtime`**.
The list will be formatted like `(year month day nil nil)`.

```el
no examples :(
```


### Timestamp (Standard)

#### om-timestamp-get-start-time `(timestamp)`

Return the time list for start time of **`timestamp`** node.
The return value will be a list as specified by the `time` argument in
[`om-build-timestamp!`](#om-build-timestamp-start-key-end-active-repeater-warning-post-blank).

```el
;; Given the following contents:
; [2019-01-01 Tue]

(->> (om-parse-this-object)
     (om-timestamp-get-start-time))
 ;; => '(2019 1 1 nil nil)

;; Given the following contents:
; [2019-01-01 Tue]--[2019-01-02 Wed]

(->> (om-parse-this-object)
     (om-timestamp-get-start-time))
 ;; => '(2019 1 1 nil nil)

;; Given the following contents:
; [2019-01-01 Tue 00:00-12:00]

(->> (om-parse-this-object)
     (om-timestamp-get-start-time))
 ;; => '(2019 1 1 0 0)

```

#### om-timestamp-get-end-time `(timestamp)`

Return the end time list for end time of **`timestamp`** or nil if not a range.
The return value will be a list as specified by the `time` argument in
[`om-build-timestamp!`](#om-build-timestamp-start-key-end-active-repeater-warning-post-blank).

```el
;; Given the following contents:
; [2019-01-01 Tue]

(->> (om-parse-this-object)
     (om-timestamp-get-end-time))
 ;; => nil

;; Given the following contents:
; [2019-01-01 Tue]--[2019-01-02 Wed]

(->> (om-parse-this-object)
     (om-timestamp-get-end-time))
 ;; => '(2019 1 2 nil nil)

;; Given the following contents:
; [2019-01-01 Tue 00:00-12:00]

(->> (om-parse-this-object)
     (om-timestamp-get-end-time))
 ;; => '(2019 1 1 12 0)

```

#### om-timestamp-get-range `(timestamp)`

Return the range of **`timestamp`** node in seconds as an integer.
If non-ranged, this function will return 0. If ranged but
the start time is in the future relative to end the time, return
a negative integer.

```el
;; Given the following contents:
; [2019-01-01 Tue]

(->> (om-parse-this-object)
     (om-timestamp-get-range))
 ;; => 0

;; Given the following contents:
; [2019-01-01 Tue]--[2019-01-02 Wed]

(->> (om-parse-this-object)
     (om-timestamp-get-range))
 ;; => 86400

;; Given the following contents:
; [2019-01-01 Tue 00:00-12:00]

(->> (om-parse-this-object)
     (om-timestamp-get-range))
 ;; => 43200

```

#### om-timestamp-is-active `(timestamp)`

Return t if **`timestamp`** node is active.

```el
;; Given the following contents:
; <2019-01-01 Tue>

(->> (om-parse-this-object)
     (om-timestamp-is-active))
 ;; => t

;; Given the following contents:
; [2019-01-01 Tue]

(->> (om-parse-this-object)
     (om-timestamp-is-active))
 ;; => nil

```

#### om-timestamp-is-ranged `(timestamp)`

Return t if **`timestamp`** node is ranged.

```el
;; Given the following contents:
; [2019-01-01 Tue]--[2019-01-02 Wed]

(->> (om-parse-this-object)
     (om-timestamp-is-ranged))
 ;; => t

;; Given the following contents:
; [2019-01-01 Tue 00:00-12:00]

(->> (om-parse-this-object)
     (om-timestamp-is-ranged))
 ;; => t

;; Given the following contents:
; [2019-01-01 Tue]

(->> (om-parse-this-object)
     (om-timestamp-is-ranged))
 ;; => nil

```

#### om-timestamp-range-contains-p `(unixtime timestamp)`

Return t if **`unixtime`** is between start and end time of **`timestamp`** node.
The boundaries are inclusive. If **`timestamp`** has a range of zero, then
only return t if **`unixtime`** is the same as **`timestamp`**. **`timestamp`** will be
interpreted according to the localtime of the operating system.

```el
;; Given the following contents:
; [2019-01-01 Tue 00:00]

(let ((ut (om-time-to-unixtime '(2019 1 1 0 0))))
  (->> (om-parse-this-object)
       (om-timestamp-range-contains-p ut)))
 ;; => t

(let ((ut (om-time-to-unixtime '(2019 1 1 0 30))))
  (->> (om-parse-this-object)
       (om-timestamp-range-contains-p ut)))
 ;; => nil

;; Given the following contents:
; [2019-01-01 Tue 00:00-01:00]

(let ((ut (om-time-to-unixtime '(2019 1 1 0 30))))
  (->> (om-parse-this-object)
       (om-timestamp-range-contains-p ut)))
 ;; => t

```

#### om-timestamp-set-collapsed `(flag timestamp)`

Return **`timestamp`** with collapsed set to **`flag`**.

If timestamp is ranged but not outside of one day, it may be collapsed
(**`flag`** is t) to short format like [yyyy-mm-dd xxx hh:mm-hh:mm] or
expanded (**`flag`** is nil) to long format like [yyyy-mm-dd xxx
hh:mm]--[yyyy-mm-dd xxx hh:mm]. If these conditions are not met,
return **`timestamp`** untouched regardless of **`flag`**.

Note: the default for all timestamp functions in `om.el` is to favor
collapsed format.

```el
;; Given the following contents:
; [2019-01-01 Tue 12:00-13:00]

(->> (om-parse-this-object)
     (om-timestamp-set-collapsed nil)
     (om-to-trimmed-string))
 ;; => "[2019-01-01 Tue 12:00]--[2019-01-01 Tue 13:00]"

;; Given the following contents:
; [2019-01-01 Tue 12:00-13:00]

(->> (om-parse-this-object)
     (om-timestamp-set-collapsed nil)
     (om-timestamp-set-collapsed t)
     (om-to-trimmed-string))
 ;; => "[2019-01-01 Tue 12:00-13:00]"

;; Given the following contents:
; [2019-01-01 Tue 12:00]

(->> (om-parse-this-object)
     (om-timestamp-set-collapsed nil)
     (om-to-trimmed-string))
 ;; => "[2019-01-01 Tue 12:00]"

;; Given the following contents:
; [2019-01-01 Tue]--[2019-01-02 Wed]

(->> (om-parse-this-object)
     (om-timestamp-set-collapsed nil)
     (om-to-trimmed-string))
 ;; => "[2019-01-01 Tue]--[2019-01-02 Wed]"

```

#### om-timestamp-set-start-time `(time timestamp)`

Return **`timestamp`** node with start time set to **`time`**.
**`time`** is a list analogous to the same argument specified in
[`om-build-timestamp!`](#om-build-timestamp-start-key-end-active-repeater-warning-post-blank).

```el
;; Given the following contents:
; [2019-01-02 Wed]

;; If not a range this will turn into a range by moving only the start time.
(->> (om-parse-this-object)
     (om-timestamp-set-start-time '(2019 1 1))
     (om-to-trimmed-string))
 ;; => "[2019-01-01 Tue]--[2019-01-02 Wed]"

;; Set a different time with different precision.
(->> (om-parse-this-object)
     (om-timestamp-set-start-time '(2019 1 1 10 0))
     (om-to-trimmed-string))
 ;; => "[2019-01-01 Tue 10:00]--[2019-01-02 Wed]"

;; Given the following contents:
; [2019-01-02 Wed 12:00]

;; If not a range and set within a day, use short format
(->> (om-parse-this-object)
     (om-timestamp-set-start-time '(2019 1 1 0 0))
     (om-to-trimmed-string))
 ;; => "[2019-01-01 Tue 00:00-12:00]"

```

#### om-timestamp-set-end-time `(time timestamp)`

Return **`timestamp`** node with end time set to **`time`**.
**`time`** is a list analogous to the same argument specified in
[`om-build-timestamp!`](#om-build-timestamp-start-key-end-active-repeater-warning-post-blank).

```el
;; Given the following contents:
; [2019-01-01 Tue]

;; Add the end time
(->> (om-parse-this-object)
     (om-timestamp-set-end-time '(2019 1 2))
     (om-to-trimmed-string))
 ;; => "[2019-01-01 Tue]--[2019-01-02 Wed]"

;; Given the following contents:
; [2019-01-01 Tue]--[2019-01-02 Wed]

;; Remove the end time
(->> (om-parse-this-object)
     (om-timestamp-set-end-time nil)
     (om-to-trimmed-string))
 ;; => "[2019-01-01 Tue]"

;; Given the following contents:
; [2019-01-01 Tue 12:00]

;; Use short range format
(->> (om-parse-this-object)
     (om-timestamp-set-end-time '(2019 1 1 13 0))
     (om-to-trimmed-string))
 ;; => "[2019-01-01 Tue 12:00-13:00]"

```

#### om-timestamp-set-single-time `(time timestamp)`

Return **`timestamp`** node with start and end times set to **`time`**.
**`time`** is a list analogous to the same argument specified in
[`om-build-timestamp!`](#om-build-timestamp-start-key-end-active-repeater-warning-post-blank).

```el
;; Given the following contents:
; [2019-01-01 Tue]

;; Don't make a range
(->> (om-parse-this-object)
     (om-timestamp-set-single-time '(2019 1 2))
     (om-to-trimmed-string))
 ;; => "[2019-01-02 Wed]"

;; Given the following contents:
; [2019-01-01 Tue]--[2019-01-02 Wed]

;; Output is not a range despite input being ranged
(->> (om-parse-this-object)
     (om-timestamp-set-single-time '(2019 1 3))
     (om-to-trimmed-string))
 ;; => "[2019-01-03 Thu]"

```

#### om-timestamp-set-double-time `(time1 time2 timestamp)`

Return **`timestamp`** node with start/end times set to **`time1`**/**`time2`** respectively.
**`time1`** and **`time2`** are lists analogous to the `time` argument specified in
[`om-build-timestamp!`](#om-build-timestamp-start-key-end-active-repeater-warning-post-blank).

```el
;; Given the following contents:
; [2019-01-01 Tue]

;; Make a range
(->> (om-parse-this-object)
     (om-timestamp-set-double-time '(2019 1 2)
				   '(2019 1 3))
     (om-to-trimmed-string))
 ;; => "[2019-01-02 Wed]--[2019-01-03 Thu]"

;; Given the following contents:
; [2019-01-01 Tue]--[2019-01-03 Wed]

(->> (om-parse-this-object)
     (om-timestamp-set-double-time '(2019 1 4)
				   '(2019 1 5))
     (om-to-trimmed-string))
 ;; => "[2019-01-04 Fri]--[2019-01-05 Sat]"

;; Given the following contents:
; [2019-01-01 Tue]--[2019-01-03 Wed]

(->> (om-parse-this-object)
     (om-timestamp-set-double-time '(2019 1 1 0 0)
				   '(2019 1 1 1 0))
     (om-to-trimmed-string))
 ;; => "[2019-01-01 Tue 00:00-01:00]"

```

#### om-timestamp-set-range `(range timestamp)`

Return **`timestamp`** node with range set to **`range`**.
If **`timestamp`** is ranged, keep start time the same and adjust the end
time. If not, make a new end time. The units for **`range`** are in minutes
if **`timestamp`** is in long format and days if **`timestamp`** is in short
format.

```el
;; Given the following contents:
; [2019-01-01 Tue]

;; Use days as the unit for short format
(->> (om-parse-this-object)
     (om-timestamp-set-range 1)
     (om-to-trimmed-string))
 ;; => "[2019-01-01 Tue]--[2019-01-02 Wed]"

;; Given the following contents:
; [2019-01-01 Tue 00:00]

;; Use minutes as the unit for long format
(->> (om-parse-this-object)
     (om-timestamp-set-range 3)
     (om-to-trimmed-string))
 ;; => "[2019-01-01 Tue 00:00-00:03]"

;; Given the following contents:
; [2019-01-01 Tue]--[2019-01-03 Wed]

;; Set range to 0 to remove end time
(->> (om-parse-this-object)
     (om-timestamp-set-range 0)
     (om-to-trimmed-string))
 ;; => "[2019-01-01 Tue]"

```

#### om-timestamp-set-active `(flag timestamp)`

Return **`timestamp`** node with active type if **`flag`** is t.

```el
;; Given the following contents:
; [2019-01-01 Tue]

(->> (om-parse-this-object)
     (om-timestamp-set-active t)
     (om-to-trimmed-string))
 ;; => "<2019-01-01 Tue>"

;; Given the following contents:
; <2019-01-01 Tue>

(->> (om-parse-this-object)
     (om-timestamp-set-active nil)
     (om-to-trimmed-string))
 ;; => "[2019-01-01 Tue]"

```

#### om-timestamp-shift `(n unit timestamp)`

Return **`timestamp`** node with time shifted by **`n`** **`unit`**`s.

This function will move the start and end times together; therefore
ranged inputs will always output ranged timestamps and same for
non-ranged. To move the start and end time independently, use
[`om-timestamp-shift-start`](#om-timestamp-shift-start-n-unit-timestamp) or [`om-timestamp-shift-end`](#om-timestamp-shift-end-n-unit-timestamp).

**`n`** is a positive or negative integer and **`unit`** is one of `minute`,
`hour`, `day`, `month`, or `year`. Overflows will wrap around
transparently; for instance, supplying `minute` for **`unit`** and 90 for **`n`**
will increase the hour property by 1 and the minute property by 30.

```el
;; Given the following contents:
; [2019-01-01 Tue 12:00]

;; Change each unit, and wrap around to the next unit as needed.
(->> (om-parse-this-object)
     (om-timestamp-shift 30 'minute)
     (om-to-trimmed-string))
 ;; => "[2019-01-01 Tue 12:30]"

(->> (om-parse-this-object)
     (om-timestamp-shift 13 'month)
     (om-to-trimmed-string))
 ;; => "[2020-02-01 Sat 12:00]"

;; Given the following contents:
; [2019-01-01 Tue]

;; Error when shifting hour/minute in short format
(->> (om-parse-this-object)
     (om-timestamp-shift 30 'minute)
     (om-to-trimmed-string))
Error

```

#### om-timestamp-shift-start `(n unit timestamp)`

Return **`timestamp`** node with start time shifted by **`n`** **`unit`**`s.

**`n`** and **`unit`** behave the same as those in [`om-timestamp-shift`](#om-timestamp-shift-n-unit-timestamp).

If **`timestamp`** is not range, the output will be a ranged timestamp with
the shifted start time and the end time as that of **`timestamp`**. If this
behavior is not desired, use [`om-timestamp-shift`](#om-timestamp-shift-n-unit-timestamp).

```el
;; Given the following contents:
; [2019-01-01 Tue 12:00]

;; If not a range, change start time and leave implicit end time.
(->> (om-parse-this-object)
     (om-timestamp-shift-start -1 'year)
     (om-to-trimmed-string))
 ;; => "[2018-01-01 Mon 12:00]--[2019-01-01 Tue 12:00]"

(->> (om-parse-this-object)
     (om-timestamp-shift-start -1 'hour)
     (om-to-trimmed-string))
 ;; => "[2019-01-01 Tue 11:00-12:00]"

;; Given the following contents:
; [2019-01-01 Tue]--[2019-01-03 Thu]

;; Change only start time if a range
(->> (om-parse-this-object)
     (om-timestamp-shift-start 1 'day)
     (om-to-trimmed-string))
 ;; => "[2019-01-02 Wed]--[2019-01-03 Thu]"

```

#### om-timestamp-shift-end `(n unit timestamp)`

Return **`timestamp`** node with end time shifted by **`n`** **`unit`**`s.

**`n`** and **`unit`** behave the same as those in [`om-timestamp-shift`](#om-timestamp-shift-n-unit-timestamp).

If **`timestamp`** is not range, the output will be a ranged timestamp with
the shifted end time and the start time as that of **`timestamp`**. If this
behavior is not desired, use [`om-timestamp-shift`](#om-timestamp-shift-n-unit-timestamp).

```el
;; Given the following contents:
; [2019-01-01 Tue]

;; Shift implicit end time if not a range.
(->> (om-parse-this-object)
     (om-timestamp-shift-end 1 'day)
     (om-to-trimmed-string))
 ;; => "[2019-01-01 Tue]--[2019-01-02 Wed]"

;; Given the following contents:
; [2019-01-01 Tue]--[2019-01-02 Wed]

;; Move only the second time if a range.
(->> (om-parse-this-object)
     (om-timestamp-shift-end 1 'day)
     (om-to-trimmed-string))
 ;; => "[2019-01-01 Tue]--[2019-01-03 Thu]"

```

#### om-timestamp-toggle-active `(timestamp)`

Return **`timestamp`** node with its active/inactive type flipped.

```el
;; Given the following contents:
; [2019-01-01 Tue]

(->> (om-parse-this-object)
     (om-timestamp-toggle-active)
     (om-to-trimmed-string))
 ;; => "<2019-01-01 Tue>"

;; Given the following contents:
; <2019-01-01 Tue>--<2019-01-02 Wed>

(->> (om-parse-this-object)
     (om-timestamp-toggle-active)
     (om-to-trimmed-string))
 ;; => "[2019-01-01 Tue]--[2019-01-02 Wed]"

```

#### om-timestamp-truncate `(timestamp)`

Return **`timestamp`** node with start/end times forced to short format.

```el
;; Given the following contents:
; [2019-01-01 Tue]--[2019-01-02 Wed]

(->> (om-parse-this-object)
     (om-timestamp-truncate)
     (om-to-trimmed-string))
 ;; => "[2019-01-01 Tue]--[2019-01-02 Wed]"

;; Given the following contents:
; [2019-01-01 Tue 12:00]--[2019-01-02 Wed 13:00]

(->> (om-parse-this-object)
     (om-timestamp-truncate)
     (om-to-trimmed-string))
 ;; => "[2019-01-01 Tue]--[2019-01-02 Wed]"

```

#### om-timestamp-truncate-start `(timestamp)`

Return **`timestamp`** node with start time forced to short format.

```el
;; Given the following contents:
; [2019-01-01 Tue 12:00]

(->> (om-parse-this-object)
     (om-timestamp-truncate-start)
     (om-to-trimmed-string))
 ;; => "[2019-01-01 Tue]"

;; Given the following contents:
; [2019-01-01 Tue 12:00]--[2019-01-02 Wed 12:00]

(->> (om-parse-this-object)
     (om-timestamp-truncate-start)
     (om-to-trimmed-string))
 ;; => "[2019-01-01 Tue]--[2019-01-02 Wed 12:00]"

;; Given the following contents:
; [2019-01-01 Tue]

(->> (om-parse-this-object)
     (om-timestamp-truncate-start)
     (om-to-trimmed-string))
 ;; => "[2019-01-01 Tue]"

```

#### om-timestamp-truncate-end `(timestamp)`

Return **`timestamp`** node with end time forced to short format.

```el
;; Given the following contents:
; [2019-01-01 Tue]--[2019-01-02 Wed]

(->> (om-parse-this-object)
     (om-timestamp-truncate-end)
     (om-to-trimmed-string))
 ;; => "[2019-01-01 Tue]--[2019-01-02 Wed]"

;; Given the following contents:
; [2019-01-01 Tue 12:00]--[2019-01-02 Wed 13:00]

(->> (om-parse-this-object)
     (om-timestamp-truncate-end)
     (om-to-trimmed-string))
 ;; => "[2019-01-01 Tue 12:00]--[2019-01-02 Wed]"

;; Given the following contents:
; [2019-01-01 Tue 12:00]

(->> (om-parse-this-object)
     (om-timestamp-truncate-end)
     (om-to-trimmed-string))
 ;; => "[2019-01-01 Tue 12:00]"

```


### Timestamp (diary)

#### om-timestamp-diary-set-value `(form timestamp-diary)`

Return **`timestamp-diary`** node with value set to **`form`**.
`timestamp` must have a type `eq` to `diary`. **`form`** is a quoted list.

```el
;; Given the following contents:
; <%%(diary-float t 4 2)>

(->> (om-parse-this-object)
     (om-timestamp-diary-set-value '(diary-float 1 3 2))
     (om-to-string))
 ;; => "<%%(diary-float 1 3 2)>"

```


## Branch/Child Manipulation


Set, get, and map the children of branch nodes.


### Polymorphic

#### om-children-contain-point `(point branch-node)`

Return t if **`point`** is within the boundaries of **`branch-node`**`s children.

```el
;; Given the following contents:
; * headline
; findme

(->> (om-parse-this-headline)
     (om-children-contain-point 2))
 ;; => nil

(->> (om-parse-this-headline)
     (om-children-contain-point 15))
 ;; => t

```

#### om-get-children `(branch-node)`

Return the children of **`branch-node`** as a list.

```el
;; Given the following contents:
; /this/ is a *paragraph*

;; Return child nodes for branch nodes
(->> (om-parse-this-element)
     (om-get-children)
     (-map (function om-get-type)))
 ;; => '(italic plain-text bold)

;; Given the following contents:
; * headline

;; Return nil if no children
(->> (om-parse-this-subtree)
     (om-get-children)
     (-map (function om-get-type)))
 ;; => nil

```

#### om-set-children `(children branch-node)`

Return **`branch-node`** with its children set to **`children`**.
**`children`** is a list of nodes; the types permitted in this list depend
on the type of `node`.

```el
;; Given the following contents:
; /this/ is a *paragraph*

;; Set children for branch object
(->> (om-parse-this-element)
     (om-set-children (list "this is lame"))
     (om-to-trimmed-string))
 ;; => "this is lame"

;; Given the following contents:
; * headline

;; Set children for branch element nodes
(->> (om-parse-this-subtree)
     (om-set-children (list (om-build-headline! :title-text "only me" :level 2)))
     (om-to-trimmed-string))
 ;; => "* headline
 ;      ** only me"

```

#### om-map-children `(fun branch-node)`

Return **`branch-node`** with **`fun`** applied to its children.
**`fun`** is a unary function that takes the current list of children and
returns a modified list of children.

```el
;; Given the following contents:
; /this/ is a *paragraph*

(->> (om-parse-this-element)
     (om-map-children (lambda (objs)
			(append objs (list " ...yeah"))))
     (om-to-trimmed-string))
 ;; => "/this/ is a *paragraph* ...yeah"

;; Given the following contents:
; * headline
; ** subheadline

(->> (om-parse-this-subtree)
     (om-map-children* (--map (om-shift-property :level 1 it)
			      it))
     (om-to-trimmed-string))
 ;; => "* headline
 ;      *** subheadline"

```

#### om-is-childless `(branch-node)`

Return t if **`branch-node`** is empty.

```el
;; Given the following contents:
; * dummy
; filled with useless knowledge

(->> (om-parse-this-headline)
     (om-is-childless))
 ;; => nil

;; Given the following contents:
; * dummy

(->> (om-parse-this-headline)
     (om-is-childless))
 ;; => t

```


### Object Nodes

#### om-unwrap `(object-node)`

Return the children of **`object-node`** as a secondary string.
If **`object-node`** is a plain-text node, wrap it in a list and return.
Else add the post-blank property of **`object-node`** to the last member
of its children and return children as a secondary string.

```el
;; Given the following contents:
; _1 *2* 3 */4/* 5 /6/_

;; Remove the outer underline formatting
(->> (om-parse-this-object)
     (om-unwrap)
     (apply (function om-build-paragraph))
     (om-to-trimmed-string))
 ;; => "1 *2* 3 */4/* 5 /6/"

```

#### om-unwrap-types-deep `(types object-node)`

Return the children of **`object-node`** as a secondary string.
If **`object-node`** is a plain-text node, wrap it in a list and return.
Else recursively descend into the children of **`object-node`** and splice
the children of nodes with type in **`types`** in place of said node and
return the result as a secondary string.

```el
;; Given the following contents:
; _1 *2* 3 */4/* 5 /6/_

;; Remove bold formatting at any level
(->> (om-parse-this-object)
     (om-unwrap-types-deep '(bold))
     (apply (function om-build-paragraph))
     (om-to-trimmed-string))
 ;; => "_1 2 3 /4/ 5 /6/_"

```

#### om-unwrap-deep `(object-node)`

Return the children of **`object-node`** as plain-text wrapped in a list.

```el
;; Given the following contents:
; _1 *2* 3 */4/* 5 /6/_

;; Remove all formatting
(->> (om-parse-this-object)
     (om-unwrap-deep)
     (apply (function om-build-paragraph))
     (om-to-trimmed-string))
 ;; => "1 2 3 4 5 6"

```


### Secondary Strings

#### om-flatten `(secondary-string)`

Return **`secondary-string`** with its first level unwrapped.
The unwrap operation will be done with [`om-unwrap`](#om-unwrap-object-node).

```el
;; Given the following contents:
; This (1 *2* 3 */4/* 5 /6/) is randomly formatted

;; Remove first level of formatting
(->> (om-parse-this-element)
     (om-map-children (function om-flatten))
     (om-to-trimmed-string))
 ;; => "This (1 2 3 /4/ 5 6) is randomly formatted"

```

#### om-flatten-types-deep `(types secondary-string)`

Return **`secondary-string`** with object nodes in **`types`** unwrapped.
The unwrap operation will be done with [`om-unwrap-types-deep`](#om-unwrap-types-deep-types-object-node).

```el
;; Given the following contents:
; This (1 *2* 3 */4/* 5 /6/) is randomly formatted

;; Remove italic formatting at any level
(->> (om-parse-this-element)
     (om-map-children* (om-flatten-types-deep '(italic)
					      it))
     (om-to-trimmed-string))
 ;; => "This (1 *2* 3 *4* 5 6) is randomly formatted"

```

#### om-flatten-deep `(secondary-string)`

Return **`secondary-string`** with all object nodes unwrapped to plain-text.
The unwrap operation will be done with [`om-unwrap-deep`](#om-unwrap-deep-object-node).

```el
;; Given the following contents:
; This (1 *2* 3 */4/* 5 /6/) is randomly formatted

;; Remove italic formatting at any level
(->> (om-parse-this-element)
     (om-map-children (function om-flatten-deep))
     (om-to-trimmed-string))
 ;; => "This (1 2 3 4 5 6) is randomly formatted"

```


### Headline

#### om-headline-get-node-properties `(headline)`

Return a list of node-properties nodes in **`headline`** or nil if none.

```el
;; Given the following contents:
; * headline
; :PROPERTIES:
; :Effort:   1:00
; :END:

(->> (om-parse-this-headline)
     (om-headline-get-node-properties)
     (-map (function om-to-trimmed-string)))
 ;; => '(":Effort:   1:00")

;; Given the following contents:
; * headline

(->> (om-parse-this-headline)
     (om-headline-get-node-properties)
     (-map (function om-to-trimmed-string)))
 ;; => nil

```

#### om-headline-get-properties-drawer `(headline)`

Return the properties drawer node in **`headline`**.

If multiple are present (there shouldn't be) the first will be
returned.

```el
;; Given the following contents:
; * headline
; :PROPERTIES:
; :Effort:   1:00
; :END:

(->> (om-parse-this-headline)
     (om-headline-get-properties-drawer)
     (om-to-trimmed-string))
 ;; => ":PROPERTIES:
 ;      :Effort:   1:00
 ;      :END:"

;; Given the following contents:
; * headline

(->> (om-parse-this-headline)
     (om-headline-get-properties-drawer)
     (om-to-trimmed-string))
 ;; => ""

```

#### om-headline-get-planning `(headline)`

Return the planning node in **`headline`** or nil if none.

```el
;; Given the following contents:
; * headline
; CLOSED: [2019-01-01 Tue]

(->> (om-parse-this-headline)
     (om-headline-get-planning)
     (om-to-trimmed-string))
 ;; => "CLOSED: [2019-01-01 Tue]"

;; Given the following contents:
; * headline

(->> (om-parse-this-headline)
     (om-headline-get-planning)
     (om-to-trimmed-string))
 ;; => ""

```

#### om-headline-set-planning `(planning headline)`

Return **`headline`** node with planning components set to **`planning`** node.

```el
;; Given the following contents:
; * headline

(->> (om-parse-this-headline)
     (om-headline-set-planning (om-build-planning! :closed '(2019 1 1)))
     (om-to-trimmed-string))
 ;; => "* headline
 ;      CLOSED: <2019-01-01 Tue>"

;; Given the following contents:
; * headline
; CLOSED: <2019-01-01 Tue>

(->> (om-parse-this-headline)
     (om-headline-set-planning (om-build-planning! :scheduled '(2019 1 1)))
     (om-to-trimmed-string))
 ;; => "* headline
 ;      SCHEDULED: <2019-01-01 Tue>"

;; Given the following contents:
; * headline
; CLOSED: <2019-01-01 Tue>

(->> (om-parse-this-headline)
     (om-headline-set-planning nil)
     (om-to-trimmed-string))
 ;; => "* headline"

```

#### om-headline-get-subheadlines `(headline)`

Return list of child headline nodes in **`headline`** node or nil if none.

```el
;; Given the following contents:
; * headline 1
; sectional stuff
; ** headline 2
; ** headline 3

(->> (om-parse-this-subtree)
     (om-headline-get-subheadlines)
     (-map (function om-to-trimmed-string)))
 ;; => '("** headline 2" "** headline 3")

;; Given the following contents:
; * headline 1
; sectional stuff

(->> (om-parse-this-subtree)
     (om-headline-get-subheadlines)
     (-map (function om-to-trimmed-string)))
 ;; => nil

```

#### om-headline-get-section `(headline)`

Return child section node within **`headline`** node or nil if none.

```el
;; Given the following contents:
; * headline 1
; sectional stuff
; ** headline 2
; ** headline 3

(->> (om-parse-this-subtree)
     (om-headline-get-section)
     (om-to-trimmed-string))
 ;; => "sectional stuff"

;; Given the following contents:
; * headline 1
; ** headline 2
; ** headline 3

(->> (om-parse-this-subtree)
     (om-headline-get-section)
     (om-to-trimmed-string))
 ;; => ""

```

#### om-headline-get-path `(headline)`

Return tree path of **`headline`** node.

The return value is a list of headline titles (including that from
**`headline`**) leading to the root node.

```el
;; Given the following contents:
; * one
; ** two
; *** three

(->> (om-parse-this-subtree)
     (om-headline-get-subheadlines)
     (car)
     (om-headline-get-path))
 ;; => '("one" "two")

;; Given the following contents:
; * one
; ** two
; *** three

(->> (om-parse-this-subtree)
     (om-headline-get-subheadlines)
     (car)
     (om-headline-get-subheadlines)
     (car)
     (om-headline-get-path))
 ;; => '("one" "two" "three")

```

#### om-headline-update-item-statistics `(headline)`

Return **`headline`** node with updated statistics cookie via items.

The percent/fraction will be computed as the number of checked items
over the number of items with checkboxes (non-checkbox items will
not be considered).

```el
;; Given the following contents:
; * statistically significant [/]
; - irrelevant data
; - [ ] good data
; - [X] bad data

(->> (om-parse-this-headline)
     (om-headline-update-item-statistics)
     (om-to-trimmed-string))
 ;; => "* statistically significant [1/2]
 ;      - irrelevant data
 ;      - [ ] good data
 ;      - [X] bad data"

;; Given the following contents:
; * statistically significant
; - irrelevant data
; - [ ] good data
; - [X] bad data

;; Do nothing if nothing to update
(->> (om-parse-this-headline)
     (om-headline-update-item-statistics)
     (om-to-trimmed-string))
 ;; => "* statistically significant
 ;      - irrelevant data
 ;      - [ ] good data
 ;      - [X] bad data"

```

#### om-headline-update-todo-statistics `(headline)`

Return **`headline`** node with updated statistics cookie via subheadlines.

The percent/fraction will be computed as the number of done
subheadlines over the number of todo subheadlines (eg non-todo
subheadlines will not be counted).

```el
;; Given the following contents:
; * statistically significant [/]
; ** irrelevant data
; ** TODO good data
; ** DONE bad data

(->> (om-parse-this-subtree)
     (om-headline-update-todo-statistics)
     (om-to-trimmed-string))
 ;; => "* statistically significant [1/2]
 ;      ** irrelevant data
 ;      ** TODO good data
 ;      ** DONE bad data"

;; Given the following contents:
; * statistically significant
; ** irrelevant data
; ** TODO good data
; ** DONE bad data

;; Do nothing if nothing to update
(->> (om-parse-this-subtree)
     (om-headline-update-todo-statistics)
     (om-to-trimmed-string))
 ;; => "* statistically significant
 ;      ** irrelevant data
 ;      ** TODO good data
 ;      ** DONE bad data"

```

#### om-headline-indent-subheadline `(index headline)`

Return **`headline`** node with child headline at **`index`** indented.
Unlike [`om-headline-indent-subtree`](#om-headline-indent-subtree-index-headline) this will not indent the
indented headline node's children.

```el
;; Given the following contents:
; * one
; ** two
; ** three
; *** four

(->> (om-parse-element-at 1)
     (om-headline-indent-subheadline 0)
     (om-to-trimmed-string))
Error

(->> (om-parse-element-at 1)
     (om-headline-indent-subheadline 1)
     (om-to-trimmed-string))
 ;; => "* one
 ;      ** two
 ;      *** three
 ;      *** four"

```

#### om-headline-indent-subtree `(index headline)`

Return **`headline`** node with child headline at **`index`** indented.
Unlike [`om-headline-indent-subheadline`](#om-headline-indent-subheadline-index-headline) this will also indent the
indented headline node's children.

```el
;; Given the following contents:
; * one
; ** two
; ** three
; *** four

(->> (om-parse-element-at 1)
     (om-headline-indent-subtree 1)
     (om-to-trimmed-string))
 ;; => "* one
 ;      ** two
 ;      *** three
 ;      **** four"

```

#### om-headline-unindent-subheadline `(index child-index headline)`

Return **`headline`** node with a child headline under **`index`** unindented.
The specific child headline to unindent is selected by **`child-index`**.

```el
;; Given the following contents:
; * one
; ** two
; ** three
; *** four
; *** four
; *** four

(->> (om-parse-element-at 1)
     (om-headline-unindent-subheadline 1 1)
     (om-to-trimmed-string))
 ;; => "* one
 ;      ** two
 ;      ** three
 ;      *** four
 ;      ** four
 ;      *** four"

```

#### om-headline-unindent-all-subheadlines `(index headline)`

Return **`headline`** node with all child headlines under **`index`** unindented.

```el
;; Given the following contents:
; * one
; ** two
; ** three
; *** four
; *** four
; *** four

(->> (om-parse-element-at 1)
     (om-headline-unindent-all-subheadlines 1)
     (om-to-trimmed-string))
 ;; => "* one
 ;      ** two
 ;      ** three
 ;      ** four
 ;      ** four
 ;      ** four"

```


### Plain List

#### om-plain-list-set-type `(type plain-list)`

Return **`plain-list`** node with type property set to **`type`**.
**`type`** is one of the symbols `unordered` or `ordered`.

```el
;; Given the following contents:
; - [ ] one
; - [X] two

(->> (om-parse-this-element)
     (om-plain-list-set-type 'ordered)
     (om-to-trimmed-string))
 ;; => "1. [ ] one
 ;      2. [X] two"

;; Given the following contents:
; 1. [ ] one
; 2. [X] two

(->> (om-parse-this-element)
     (om-plain-list-set-type 'unordered)
     (om-to-trimmed-string))
 ;; => "- [ ] one
 ;      - [X] two"

```

#### om-plain-list-indent-item `(index plain-list)`

Return **`plain-list`** node with child item at **`index`** indented.
Unlike `om-item-indent-item-tree` this will not indent the indented
item node's children.

```el
;; Given the following contents:
; - one
; - two
;   - three
; - four

;; It makes no sense to indent the first item
(->> (om-parse-element-at 1)
     (om-plain-list-indent-item 0)
     (om-to-trimmed-string))
Error

(->> (om-parse-element-at 1)
     (om-plain-list-indent-item 1)
     (om-to-trimmed-string))
 ;; => "- one
 ;        - two
 ;        - three
 ;      - four"

(->> (om-parse-element-at 1)
     (om-plain-list-indent-item 2)
     (om-to-trimmed-string))
 ;; => "- one
 ;      - two
 ;        - three
 ;        - four"

```

#### om-plain-list-indent-item-tree `(index plain-list)`

Return **`plain-list`** node with child item at **`index`** indented.
Unlike `om-item-indent-item` this will also indent the indented item
node's children.

```el
;; Given the following contents:
; - one
; - two
;   - three
; - four

(->> (om-parse-element-at 1)
     (om-plain-list-indent-item-tree 1)
     (om-to-trimmed-string))
 ;; => "- one
 ;        - two
 ;          - three
 ;      - four"

```

#### om-plain-list-unindent-item `(index child-index plain-list)`

Return **`plain-list`** node with a child item under **`index`** unindented.
The specific child item to unindent is selected by **`child-index`**.

```el
;; Given the following contents:
; - one
; - two
;   - three
;   - three
;   - three
; - four

(->> (om-parse-element-at 1)
     (om-plain-list-unindent-item 1 0)
     (om-to-trimmed-string))
 ;; => "- one
 ;      - two
 ;      - three
 ;        - three
 ;        - three
 ;      - four"

(->> (om-parse-element-at 1)
     (om-plain-list-unindent-item 1 1)
     (om-to-trimmed-string))
 ;; => "- one
 ;      - two
 ;        - three
 ;      - three
 ;        - three
 ;      - four"

(->> (om-parse-element-at 1)
     (om-plain-list-unindent-item 2 1)
     (om-to-trimmed-string))
 ;; => "- one
 ;      - two
 ;        - three
 ;        - three
 ;        - three
 ;      - four"

```

#### om-plain-list-unindent-all-items `(index plain-list)`

Return **`plain-list`** node with all child items under **`index`** unindented.

```el
;; Given the following contents:
; - one
; - two
;   - three
;   - three
;   - three
; - four

(->> (om-parse-element-at 1)
     (om-plain-list-unindent-all-items 1)
     (om-to-trimmed-string))
 ;; => "- one
 ;      - two
 ;      - three
 ;      - three
 ;      - three
 ;      - four"

(->> (om-parse-element-at 1)
     (om-plain-list-unindent-all-items 2)
     (om-to-trimmed-string))
 ;; => "- one
 ;      - two
 ;        - three
 ;        - three
 ;        - three
 ;      - four"

```


### Table

#### om-table-get-cell `(row-index column-index table)`

Return table-cell node at **`row-index`** and **`column-index`** in **`table`** node.
Rule-type rows do not count toward row indices.

```el
;; Given the following contents:
; | 1 | 2 | 3 |
; |---+---+---|
; | a | b | c |

(->> (om-parse-this-element)
     (om-table-get-cell 0 0)
     (om-get-children)
     (car))
 ;; => "1"

(->> (om-parse-this-element)
     (om-table-get-cell 1 1)
     (om-get-children)
     (car))
 ;; => "b"

(->> (om-parse-this-element)
     (om-table-get-cell -1 -1)
     (om-get-children)
     (car))
 ;; => "c"

```

#### om-table-delete-column `(column-index table)`

Return **`table`** node with column at **`column-index`** deleted.

```el
;; Given the following contents:
; | a | b |
; |---+---|
; | c | d |

(->> (om-parse-this-element)
     (om-table-delete-column 0)
     (om-to-trimmed-string))
 ;; => "| b |
 ;      |---|
 ;      | d |"

(->> (om-parse-this-element)
     (om-table-delete-column 1)
     (om-to-trimmed-string))
 ;; => "| a |
 ;      |---|
 ;      | c |"

(->> (om-parse-this-element)
     (om-table-delete-column -1)
     (om-to-trimmed-string))
 ;; => "| a |
 ;      |---|
 ;      | c |"

```

#### om-table-delete-row `(row-index table)`

Return **`table`** node with row at **`row-index`** deleted.

```el
;; Given the following contents:
; | a | b |
; |---+---|
; | c | d |

(->> (om-parse-this-element)
     (om-table-delete-row 0)
     (om-to-trimmed-string))
 ;; => "|---+---|
 ;      | c | d |"

(->> (om-parse-this-element)
     (om-table-delete-row 1)
     (om-to-trimmed-string))
 ;; => "| a | b |
 ;      | c | d |"

(->> (om-parse-this-element)
     (om-table-delete-row -1)
     (om-to-trimmed-string))
 ;; => "| a | b |
 ;      |---+---|"

```

#### om-table-insert-column! `(column-index column-text table)`

Return **`table`** node with **`column-text`** inserted at **`column-index`**.

**`column-index`** is the index of the column and **`column-text`** is a list of
strings to be made into table-cells to be inserted following the same
syntax as [`om-build-table-cell!`](#om-build-table-cell-string).

```el
;; Given the following contents:
; | a | b |
; |---+---|
; | c | d |

(->> (om-parse-this-element)
     (om-table-insert-column! 1 '("x" "y"))
     (om-to-trimmed-string))
 ;; => "| a | x | b |
 ;      |---+---+---|
 ;      | c | y | d |"

(->> (om-parse-this-element)
     (om-table-insert-column! -1 '("x" "y"))
     (om-to-trimmed-string))
 ;; => "| a | b | x |
 ;      |---+---+---|
 ;      | c | d | y |"

```

#### om-table-insert-row! `(row-index row-text table)`

Return **`table`** node with **`row-text`** inserted at **`row-index`**.

**`row-index`** is the index of the column and **`row-text`** is a list of strings
to be made into table-cells to be inserted following the same syntax
as [`om-build-table-row!`](#om-build-table-row-row-list).

```el
;; Given the following contents:
; | a | b |
; |---+---|
; | c | d |

(->> (om-parse-this-element)
     (om-table-insert-row! 1 '("x" "y"))
     (om-to-trimmed-string))
 ;; => "| a | b |
 ;      | x | y |
 ;      |---+---|
 ;      | c | d |"

(->> (om-parse-this-element)
     (om-table-insert-row! 2 '("x" "y"))
     (om-to-trimmed-string))
 ;; => "| a | b |
 ;      |---+---|
 ;      | x | y |
 ;      | c | d |"

(->> (om-parse-this-element)
     (om-table-insert-row! -1 '("x" "y"))
     (om-to-trimmed-string))
 ;; => "| a | b |
 ;      |---+---|
 ;      | c | d |
 ;      | x | y |"

```

#### om-table-replace-cell! `(row-index column-index cell-text table)`

Return **`table`** node with a table-cell node replaced by **`cell-text`**.

If **`cell-text`** is a string, it will replace the children of the
table-cell at **`row-index`** and **`column-index`** in **`table`**. **`cell-text`** will be
processed the same as the argument given to [`om-build-table-cell!`](#om-build-table-cell-string).

If **`cell-text`** is nil, it will set the cell to an empty string.

```el
;; Given the following contents:
; | 1 | 2 |
; |---+---|
; | a | b |

(->> (om-parse-this-element)
     (om-table-replace-cell! 0 0 "2")
     (om-to-trimmed-string))
 ;; => "| 2 | 2 |
 ;      |---+---|
 ;      | a | b |"

(->> (om-parse-this-element)
     (om-table-replace-cell! 0 0 nil)
     (om-to-trimmed-string))
 ;; => "|   | 2 |
 ;      |---+---|
 ;      | a | b |"

(->> (om-parse-this-element)
     (om-table-replace-cell! -1 -1 "B")
     (om-to-trimmed-string))
 ;; => "| 1 | 2 |
 ;      |---+---|
 ;      | a | B |"

```

#### om-table-replace-column! `(column-index column-text table)`

Return **`table`** node with the column at **`column-index`** replaced by **`column-text`**.

If **`column-text`** is a list of strings, it will replace the table-cells
at **`column-index`**. Each member of **`column-text`** will be processed the
same as the argument given to [`om-build-table-cell!`](#om-build-table-cell-string).

If **`column-text`** is nil, it will clear all cells at **`column-index`**.

```el
;; Given the following contents:
; | a | b |
; |---+---|
; | c | d |

(->> (om-parse-this-element)
     (om-table-replace-column! 0 '("A" "B"))
     (om-to-trimmed-string))
 ;; => "| A | b |
 ;      |---+---|
 ;      | B | d |"

(->> (om-parse-this-element)
     (om-table-replace-column! 0 nil)
     (om-to-trimmed-string))
 ;; => "|   | b |
 ;      |---+---|
 ;      |   | d |"

(->> (om-parse-this-element)
     (om-table-replace-column! -1 '("A" "B"))
     (om-to-trimmed-string))
 ;; => "| a | A |
 ;      |---+---|
 ;      | c | B |"

```

#### om-table-replace-row! `(row-index row-text table)`

Return **`table`** node with the row at **`row-index`** replaced by **`row-text`**.

If **`row-text`** is a list of strings, it will replace the cells at
**`row-index`**. Each member of **`row-text`** will be processed the same as
the argument given to [`om-build-table-row!`](#om-build-table-row-row-list).

If **`row-text`** is nil, it will clear all cells at **`row-index`**.

```el
;; Given the following contents:
; | a | b |
; |---+---|
; | c | d |

(->> (om-parse-this-element)
     (om-table-replace-row! 0 '("A" "B"))
     (om-to-trimmed-string))
 ;; => "| A | B |
 ;      |---+---|
 ;      | c | d |"

(->> (om-parse-this-element)
     (om-table-replace-row! 0 nil)
     (om-to-trimmed-string))
 ;; => "|   |   |
 ;      |---+---|
 ;      | c | d |"

(->> (om-parse-this-element)
     (om-table-replace-row! -1 '("A" "B"))
     (om-to-trimmed-string))
 ;; => "| a | b |
 ;      |---+---|
 ;      | A | B |"

```


## Node Matching


Use pattern-matching to selectively perform operations on nodes in trees.

#### om-match `(pattern node)`

Return a list of child nodes matching **`pattern`** in **`node`**.

**`pattern`** is a list like `([slicer [arg1] [arg2]] [pnode] sub1 [sub2 ...])`.

`slicer` is an optional prefix to the pattern describing how many
and which matches to return. If not given, all matches are returned.
Possible values are:

- `:first` - return the first match
- `:last` - return the last match
- `:nth` `arg1` - return the nth match where `arg1` is an integer denoting
    the index to return (starting at 0). `arg1` may be a negative number
    to start counting at the end of the match list, in which case -1 is
    the last index. Using 0 and -1 for `arg1` is equivalent to using
    `:first` and `:last` respectively
- `:sub` `arg1` `arg2` - return a sublist between indices `arg1` and `arg2`.
    `arg1` may not be greater than `arg2`, and both must either be
    non-negative integers or negative integers. In the case of negative
    integers, the indices refer to the same counterparts as described in
    `:nth`. If `arg1` and `arg2` are equal, this slicer has the same
    behavior as `:nth`.

`pnode` is an optional literal node to be matched. If given, the
matching process will start within `pnode` wherever it happens to be in
**`node`**. This is useful for 'reusing' previous matches without repeating
the same pattern.

`subx` denotes subpatterns that that match nodes in the parse tree.
Subpatterns may either be wildcards or conditions.

Conditions match exactly one level of the node tree being searched
based on the node's type (the symbol returned by [`om-get-type`](#om-get-type-node)),
properties (the value returned by [`om-get-property`](#om-get-property-prop-node) for a valid
property keyword), and index (the position of the node in the list
returned by [`om-get-children`](#om-get-children-branch-node)). For index, both left indices (where
zero refers to the left end of the list) and right indices (where -1
refers to the right end of the list) are understood. Conditions may
either be atomic or compound, where compound conditions are themselves
composed of atomic or compound conditions.

The types of atomic conditions are:

- `type` - match when the node's type is `eq` to `type` (a symbol)
- `index` - match when the node's index is `=` to `index` (an integer)
- `(op index)` - match when `(op node-index index)` returns t. `op` is
    one of `<`, `>`, `<=`, or `>=` and `node-index` is the index of the
    node being evaluated
- `(prop val)` - match nodes whose property `prop` (a keyword) is `equal`
    to `val`; `val` is obtained by evaluating [`om-get-property`](#om-get-property-prop-node) with `prop`
    and the current node; if `prop` is invalid, an error will be thrown
- `(:pred pred)` - match when `pred` evaluates to t; `pred` is a symbol for
    a unary function that takes the current node as its argument

Compound conditions start with an operator followed by their component
conditions. In the syntax below, `cx` refers to a condition. The types
of compound conditions are:

- `(:and c1 c2 [c3 ...])` - match when all conditions are true
- `(:or c1 c2 [c3 ...])` - match when at least one condition is true
- `(:not c)` - match when condition is not true

In addition to conditions, `subx` may be a wildcard keyword to match
nodes independent of their type, properties, and index. The types of
wildcards are:

- `:any` - always match exactly one node
- `:many` `sub` - match `sub` to nodes that are zero or more levels down
    in the tree; `sub` is a subpattern of either a condition or the `:any`
    wildcard; only one subpattern is allowed after `:many`
- `:many!` `sub` - like `:many` but do not match within other matches

```el
;; Given the following contents:
; * headline 1
; ** TODO headline 2
; stuff
; - item 1
; - item 2
; - item 3
; ** DONE headline 3
; - item 4
; - item 5
; - item 6
; ** TODO COMMENT headline 4
; - item 7
; - item 8
; - item 9

;; Match items (excluding the first) in headlines that are marked "TODO" and not
;; commented. The :many keyword matches the section and plain-list nodes holding
;; the items.
(->> (om-parse-this-subtree)
     (om-match '((:and (:todo-keyword "TODO")
			     (:commentedp nil))
		       :many (:and item (> 0))))
     (-map (function om-to-trimmed-string)))
 ;; => '("- item 2" "- item 3")

;; Given the following contents:
; *one* *two* *three* *four* *five* *six*

;; Return all bold nodes
(->> (om-parse-this-element)
     (om-match '(bold))
     (-map (function om-to-trimmed-string)))
 ;; => '("*one*" "*two*" "*three*" "*four*" "*five*" "*six*")

;; Return first bold node
(->> (om-parse-this-element)
     (om-match '(:first bold))
     (-map (function om-to-trimmed-string)))
 ;; => '("*one*")

;; Return last bold node
(->> (om-parse-this-element)
     (om-match '(:last bold))
     (-map (function om-to-trimmed-string)))
 ;; => '("*six*")

;; Return a select bold node
(->> (om-parse-this-element)
     (om-match '(:nth 2 bold))
     (-map (function om-to-trimmed-string)))
 ;; => '("*three*")

;; Return a sublist of matched bold nodes
(->> (om-parse-this-element)
     (om-match '(:sub 1 3 bold))
     (-map (function om-to-trimmed-string)))
 ;; => '("*two*" "*three*" "*four*")

```

#### om-match-delete `(pattern node)`

Return **`node`** without children matching **`pattern`**.

**`pattern`** follows the same rules as [`om-match`](#om-match-pattern-node).

```el
;; Given the following contents:
; * headline one
; ** headline two
; ** headline three
; ** headline four

;; Selectively delete headlines
(->> (om-parse-this-subtree)
     (om-match-delete '(headline))
     (om-to-trimmed-string))
 ;; => "* headline one"

(->> (om-parse-this-subtree)
     (om-match-delete '(:first headline))
     (om-to-trimmed-string))
 ;; => "* headline one
 ;      ** headline three
 ;      ** headline four"

(->> (om-parse-this-subtree)
     (om-match-delete '(:last headline))
     (om-to-trimmed-string))
 ;; => "* headline one
 ;      ** headline two
 ;      ** headline three"

```

#### om-match-extract `(pattern node)`

Remove nodes matching **`pattern`** from **`node`**.
Return cons cell where the car is a list of all removed nodes and
the cdr is the modified **`node`**.

**`pattern`** follows the same rules as [`om-match`](#om-match-pattern-node).

```el
;; Given the following contents:
; pull me /under/

(--> (om-parse-this-element)
     (om-match-extract '(:many italic)
		       it)
     (cons (-map (function om-to-trimmed-string)
		 (car it))
	   (om-to-trimmed-string (cdr it))))
 ;; => '(("/under/") . "pull me")

```

#### om-match-map `(pattern fun node)`

Return **`node`** with **`fun`** applied to children matching **`pattern`**.
**`fun`** is a unary function that takes a node and returns a new node
which will replace the original.

**`pattern`** follows the same rules as [`om-match`](#om-match-pattern-node).

```el
;; Given the following contents:
; * headline one
; ** TODO headline two
; ** headline three
; ** headline four

;; Selectively mark headlines as DONE
(->> (om-parse-this-subtree)
     (om-match-map '(headline)
       (lambda (it)
	 (om-set-property :todo-keyword "DONE" it)))
     (om-to-trimmed-string))
 ;; => "* headline one
 ;      ** DONE headline two
 ;      ** DONE headline three
 ;      ** DONE headline four"

(->> (om-parse-this-subtree)
     (om-match-map* '(:first headline)
       (om-set-property :todo-keyword "DONE" it))
     (om-to-trimmed-string))
 ;; => "* headline one
 ;      ** DONE headline two
 ;      ** headline three
 ;      ** headline four"

(->> (om-parse-this-subtree)
     (om-match-map '(:last headline)
       (-partial (function om-set-property)
		 :todo-keyword "DONE"))
     (om-to-trimmed-string))
 ;; => "* headline one
 ;      ** TODO headline two
 ;      ** headline three
 ;      ** DONE headline four"

;; Given the following contents:
; * headline
; :PROPERTIES:
; :Effort:   0:30
; :END:

;; Match the literal property-drawer node and map the node-property inside if
;; the property-drawer exists
(let ((hl (om-parse-this-headline)))
  (-if-let (pd (om-headline-get-properties-drawer hl))
      (->> hl (om-match-map* (\` ((\, pd)
				  node-property))
		(om-set-property :value "1:30" it))
	   (om-to-trimmed-string))
    (print "...or do something else if no drawer")))
 ;; => "* headline
 ;      :PROPERTIES:
 ;      :Effort:   1:30
 ;      :END:"

```

#### om-match-mapcat `(pattern fun node)`

Return **`node`** with **`fun`** applied to children matching **`pattern`**.
**`fun`** is a unary function that takes a node and returns a list of new
nodes which will be spliced in place of the original node.

**`pattern`** follows the same rules as [`om-match`](#om-match-pattern-node).

```el
;; Given the following contents:
; * one
; ** two

(->> (om-parse-this-subtree)
     (om-match-mapcat* '(:first headline)
       (list (om-build-headline! :title-text "1.5" :level 2)
	     it))
     (om-to-trimmed-string))
 ;; => "* one
 ;      ** 1.5
 ;      ** two"

```

#### om-match-replace `(pattern node* node)`

Return **`node`** with **`node*`** in place of children matching **`pattern`**.

**`pattern`** follows the same rules as [`om-match`](#om-match-pattern-node).

```el
;; Given the following contents:
; *1* 2 *3* 4 *5* 6 *7* 8 *9* 10

(->> (om-parse-this-element)
     (om-match-replace '(:many bold)
       (om-build-bold :post-blank 1 "0"))
     (om-to-trimmed-string))
 ;; => "*0* 2 *0* 4 *0* 6 *0* 8 *0* 10"

```

#### om-match-insert-before `(pattern node* node)`

Return **`node`** with **`node*`** inserted before children matching **`pattern`**.

**`pattern`** follows the same rules as [`om-match`](#om-match-pattern-node).

```el
;; Given the following contents:
; * one
; ** two
; ** three

(->> (om-parse-this-subtree)
     (om-match-insert-before '(headline)
       (om-build-headline! :title-text "new" :level 2))
     (om-to-trimmed-string))
 ;; => "* one
 ;      ** new
 ;      ** two
 ;      ** new
 ;      ** three"

```

#### om-match-insert-after `(pattern node* node)`

Return **`node`** with **`node*`** inserted after children matching **`pattern`**.

**`pattern`** follows the same rules as [`om-match`](#om-match-pattern-node).

```el
;; Given the following contents:
; * one
; ** two
; ** three

(->> (om-parse-this-subtree)
     (om-match-insert-after '(headline)
       (om-build-headline! :title-text "new" :level 2))
     (om-to-trimmed-string))
 ;; => "* one
 ;      ** two
 ;      ** new
 ;      ** three
 ;      ** new"

```

#### om-match-insert-within `(pattern index node* node)`

Return **`node`** with **`node*`** inserted at **`index`** in children matching **`pattern`**.

**`pattern`** follows the same rules as [`om-match`](#om-match-pattern-node) with the exception
that **`pattern`** may be nil. In this case **`node*`** will be inserted at **`index`**
in the immediate, top level children of **`node`**.

```el
;; Given the following contents:
; * one
; ** two
; ** three

(->> (om-parse-this-subtree)
     (om-match-insert-within '(headline)
	 0 (om-build-headline! :title-text "new" :level 3))
     (om-to-trimmed-string))
 ;; => "* one
 ;      ** two
 ;      *** new
 ;      ** three
 ;      *** new"

;; The nil pattern denotes top-level element
(->> (om-parse-this-subtree)
     (om-match-insert-within nil 1 (om-build-headline! :title-text "new" :level 2))
     (om-to-trimmed-string))
 ;; => "* one
 ;      ** two
 ;      ** new
 ;      ** three"

```

#### om-match-splice `(pattern nodes* node)`

Return **`node`** with **`nodes*`** spliced in place of children matching **`pattern`**.
**`nodes*`** is a list of nodes.

**`pattern`** follows the same rules as [`om-match`](#om-match-pattern-node).

```el
;; Given the following contents:
; * one
; ** two
; ** three

(let ((L (list (om-build-headline! :title-text "new0" :level 2)
	       (om-build-headline! :title-text "new1" :level 2))))
  (->> (om-parse-this-subtree)
       (om-match-splice '(0)
	 L)
       (om-to-trimmed-string)))
 ;; => "* one
 ;      ** new0
 ;      ** new1
 ;      ** three"

```

#### om-match-splice-before `(pattern nodes* node)`

Return **`node`** with **`nodes*`** spliced before children matching **`pattern`**.
**`nodes*`** is a list of nodes.

**`pattern`** follows the same rules as [`om-match`](#om-match-pattern-node).

```el
;; Given the following contents:
; * one
; ** two
; ** three

(let ((L (list (om-build-headline! :title-text "new0" :level 2)
	       (om-build-headline! :title-text "new1" :level 2))))
  (->> (om-parse-this-subtree)
       (om-match-splice-before '(0)
	 L)
       (om-to-trimmed-string)))
 ;; => "* one
 ;      ** new0
 ;      ** new1
 ;      ** two
 ;      ** three"

```

#### om-match-splice-after `(pattern nodes* node)`

Return **`node`** with **`nodes*`** spliced after children matching **`pattern`**.
**`nodes*`** is a list of nodes.

**`pattern`** follows the same rules as [`om-match`](#om-match-pattern-node).

```el
;; Given the following contents:
; * one
; ** two
; ** three

(let ((L (list (om-build-headline! :title-text "new0" :level 2)
	       (om-build-headline! :title-text "new1" :level 2))))
  (->> (om-parse-this-subtree)
       (om-match-splice-after '(0)
	 L)
       (om-to-trimmed-string)))
 ;; => "* one
 ;      ** two
 ;      ** new0
 ;      ** new1
 ;      ** three"

```

#### om-match-splice-within `(pattern index nodes* node)`

Return **`node`** with **`nodes*`** spliced at **`index`** in children matching **`pattern`**.
**`nodes*`** is a list of nodes.

**`pattern`** follows the same rules as [`om-match`](#om-match-pattern-node) with the exception
that **`pattern`** may be nil. In this case **`nodes*`** will be inserted at **`index`**
in the immediate, top level children of **`node`**.

```el
;; Given the following contents:
; * one
; ** two
; ** three
; *** four

(let ((L (list (om-build-headline! :title-text "new0" :level 3)
	       (om-build-headline! :title-text "new1" :level 3))))
  (->> (om-parse-this-subtree)
       (om-match-splice-within '(headline)
	   0 L)
       (om-to-trimmed-string)))
 ;; => "* one
 ;      ** two
 ;      *** new0
 ;      *** new1
 ;      ** three
 ;      *** new0
 ;      *** new1
 ;      *** four"

(let ((L (list (om-build-headline! :title-text "new0" :level 2)
	       (om-build-headline! :title-text "new1" :level 2))))
  (->> (om-parse-this-subtree)
       (om-match-splice-within nil 1 L)
       (om-to-trimmed-string)))
 ;; => "* one
 ;      ** two
 ;      ** new0
 ;      ** new1
 ;      ** three
 ;      *** four"

```

#### om-match-do `(pattern fun node)`

Like [`om-match-map`](#om-match-map-pattern-fun-node) but for side effects only.
**`fun`** is a unary function that has side effects and is applied to the
matches from **`node`** using **`pattern`**. This function itself returns nil.

**`pattern`** follows the same rules as [`om-match`](#om-match-pattern-node).

```el
no examples :(
```


## Buffer Side Effects


Map node manipulations into buffers.


### Insert

#### om-insert `(point node)`

Convert **`node`** to a string and insert at **`point`** in the current buffer.
Return **`node`**.

```el
;; Given the following contents:
; * one
; 

(->> (om-build-headline! :title-text "two")
     (om-insert (point-max)))
 ;; Output these buffer contents
 ;; $> "* one
 ;      * two"

;; Given the following contents:
; a *game* or a /boy/

(->> (om-build-paragraph! "we don't care if you're")
     (om-insert (point-min)))
 ;; Output these buffer contents
 ;; $> "we don't care if you're
 ;      a *game* or a /boy/"

```

#### om-insert-tail `(point node)`

Like [`om-insert`](#om-insert-point-node) but insert **`node`** at **`point`** and move to end of insertion.

```el
no examples :(
```


### Update

#### om-update `(fun node)`

Replace **`node`** in the current buffer with a new one.
**`fun`** is a unary function that takes **`node`** and returns a modified **`node`**.
This modified node is then written in place of the old node in the
current buffer.

```el
;; Given the following contents:
; * TODO win grammy

(->> (om-parse-this-headline)
     (om-update (lambda (hl)
		  (om-set-property :todo-keyword "DONE" hl))))
 ;; Output these buffer contents
 ;; $> "* DONE win grammy"

;; Given the following contents:
; * win grammy [0/0]
; - [ ] write punk song
; - [ ] get new vocalist
; - [ ] sell 2 singles

(->> (om-parse-this-headline)
     (om-update* (->> (om-match-map '(:many item)
			(function om-item-toggle-checkbox)
			it)
		      (om-headline-update-item-statistics))))
 ;; Output these buffer contents
 ;; $> "* win grammy [3/3]
 ;      - [X] write punk song
 ;      - [X] get new vocalist
 ;      - [X] sell 2 singles"

```

#### om-update-object-at `(point fun)`

Update object under **`point`** using **`fun`**.
**`fun`** takes an object and returns a modified object

```el
;; Given the following contents:
; [[http://example.com][desc]]

(om-update-object-at* (point)
  (om-set-property :path "//buymoreram.com" it))
 ;; Output these buffer contents
 ;; $> "[[http://buymoreram.com][desc]]"

```

#### om-update-element-at `(point fun)`

Update element under **`point`** using **`fun`**.
**`fun`** takes an element and returns a modified element

```el
;; Given the following contents:
; #+CALL: ktulu()

(om-update-element-at* (point)
  (om-set-properties (list :call "cthulhu" :inside-header '(:cache no)
			    :arguments '("x=4")
			    :end-header '(:results html))
		     it))
 ;; Output these buffer contents
 ;; $> "#+CALL: cthulhu[:cache no](x=4) :results html"

```

#### om-update-table-row-at `(point fun)`

Update table-row under **`point`** using **`fun`**.
**`fun`** takes an table-row and returns a modified table-row

```el
;; Given the following contents:
; | a | b |

(om-update-table-row-at* (point)
  (om-map-children* (cons (om-build-table-cell! "0")
			  it)
		    it))
 ;; Output these buffer contents
 ;; $> "| 0 | a | b |"

```

#### om-update-item-at `(point fun)`

Update item under **`point`** using **`fun`**.
**`fun`** takes an item and returns a modified item

```el
;; Given the following contents:
; - [ ] thing

(om-update-item-at* (point)
  (om-item-toggle-checkbox it))
 ;; Output these buffer contents
 ;; $> "- [X] thing"

```

#### om-update-headline-at `(point fun)`

Update headline under **`point`** using **`fun`**.
**`fun`** takes an headline and returns a modified headline

```el
;; Given the following contents:
; * TODO might get done
; * DONE no need to update

(om-update-headline-at* (point)
  (om-set-property :todo-keyword "DONE" it))
 ;; Output these buffer contents
 ;; $> "* DONE might get done
 ;      * DONE no need to update"

```

#### om-update-subtree-at `(point fun)`

Update subtree under **`point`** using **`fun`**.
**`fun`** takes an subtree and returns a modified subtree

```el
;; Given the following contents:
; * one
; ** two
; ** three
; * not updated

(om-update-subtree-at* (point)
  (om-headline-indent-subheadline 1 it))
 ;; Output these buffer contents
 ;; $> "* one
 ;      ** two
 ;      *** three
 ;      * not updated"

```

#### om-update-section-at `(point fun)`

Update section under **`point`** using **`fun`**.
**`fun`** takes an section and returns a modified section

```el
;; Given the following contents:
; #+KEY1: VAL1
; #+KEY2: VAL2
; * irrelevant headline

;; Update the top buffer section before the headlines start
(om-update-section-at* (point)
  (om-map-children* (--map (om-map-property :value (function s-downcase)
					     it)
			   it)
		    it))
 ;; Output these buffer contents
 ;; $> "#+KEY1: val1
 ;      #+KEY2: val2
 ;      * irrelevant headline"

```


### Misc

#### om-fold `(node)`

Fold the children of **`node`** if they exist.

```el
no examples :(
```

#### om-unfold `(node)`

Unfold the children of **`node`** if they exist.

```el
no examples :(
```


<!-- 1.0.0 -->

# Acknowledgements

- [@magnars](https://github.com/magnars):
[dash.el](https://github.com/magnars/dash.el) and
[s.el](https://github.com/magnars/s.el)
- Devin Townsend: Puppetry

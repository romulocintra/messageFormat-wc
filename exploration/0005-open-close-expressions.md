# Open/Close Expressions

Status: **Proposed**

<details>
	<summary>Metadata</summary>
	<dl>
		<dt>Contributors</dt>
		<dd>@eemeli</dd>
		<dd>@aphillips</dd>
		<dt>First proposed</dt>
		<dd>2023-09-05</dd>
		<dt>Pull Request</dt>
		<dd><a href="https://github.com/unicode-org/message-format-wg/pull/470">#470</a></dd>
	</dl>
</details>

## Objective

_What is this proposal trying to achieve?_

Describe the use cases and requirements for _expressions_
that enclose parts of a pattern
and develop a design that satisfies these needs.

## Background

_What context is helpful to understand this proposal?_

## Use-Cases

- Representing markup, such as HTML, as expressions in a pattern
  rather than as plain character sequences embedded in the _text_
  of a pattern. This also allows parts of a markup sequence to be
  programmable.

**_Non-markup use cases to consider (which are not addressed by the design)_**

- Represent templating language constructs so that they can pass through
  the translation process and are not exposed to translation. For example,
  Mustache templates.

- Represent translation metadata, such as XLIFF placeholders, so that
  portions of a message are "protected" or otherwise hinted in the
  translation process.

## Requirements

Be able to indicate that some identified markup applies to
a non-empty sequence of pattern parts (text, expressions).

Markup spans may be nested,
as in `<b>Bold and <i>also italic</i></b>`.

Markup spans may be improperly nested,
as in `<b>Bold <i>and</b> italic</i>`.

Markup may have localisable options,
such as `Click <a title="Link tooltip">here</a> to continue`.

As with function options,
markup options should be defined in a registry so that they can be validated.

## Constraints

Due to segmentation,
markup may be split across messages so that it opens in one and closes in another.

Following previously established consensus,
the resolution of the value of an _expression_ may only depend on its own contents,
without access to the other parts of the selected pattern.

## Proposed Design

This design relies on the recognition that the formatted output of MF2
may be further processed by other tools before presentation to a user.

Let us add _markup_ as a new type of _expression_,
in parallel with _annotation_:

```abnf
expression      = "{" [s] expression-body [s] "}"
expression-body = (operand [s annotation])
                / annotation
                / markup

annotation = (function *(s option))
           / reserved
           / private-use
markup     = (markup-start *(s option))
           / markup-end

function     = ":" name
markup-start = "+" name
markup-end   = "-" name
```

This allows for expressions like `{+b}`, `{-i}`, and `{+a title=|Link tooltip|}`.
Unlike annotations, markup expressions may not have operands.

Markup expressions do not support selection.

When formatting to a string,
markup expressions format to an empty string by default.
An implementation may customize this behaviour,
e.g. emitting XML-ish tags for each start/end expression.

When formatting to parts (as proposed in <a href="https://github.com/unicode-org/message-format-wg/pull/463">#463</a>),
markup expressions format to an object including the following properties:

- The `type` of the markup: `"start" | "end"`
- The `name` of the markup, e.g. `"b"` for `{+b}`
- For _markup-start_, the `options` with the resolved key-value pairs of the expression options

To make use of _markup_,
the message should be formatted to parts or to some other target supported by the implementation,
and the desired shape constructed from the parts.
For example, the message

```
{Click {+a title=|Link tooltip|}here{-a} to continue}
```

would format to parts as

```coffee
[
  { type: "text", value: "Click " },
  { type: "start", name: "a", options: { title: "Link tooltip" } },
  { type: "text", value: "here" },
  { type: "end", name: "a" },
  { type: "text", value: " to continue" }
]
```

and a post-MessageFormat step could then reconstruct this into an appropriate shape,
potentially merging in non-localizable attributes such as a link URL.
Rendered as React this could become:

```coffee
[
  "Click ",
  createElement("a", { href: "/next", title: "Link tooltip" }, "here"),
  " to continue"
]
```

## Alternatives Considered

_What other solutions are available?_
_How do they compare against the requirements?_
_What other properties they have?_
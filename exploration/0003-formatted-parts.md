# Formatted Parts

Status: **Proposed**

<details>
	<summary>Metadata</summary>
	<dl>
		<dt>Contributors</dt>
		<dd>@eemeli</dd>
		<dt>First proposed</dt>
		<dd>2023-08-29</dd>
		<dt>Pull Request</dt>
		<dd>#000</dd>
	</dl>
</details>

## Objective

Messages often include placeholders that,
when formatted, contain internal structure ("parts")
that the caller might want access to
for the purposes of styling, presentation, or manipulation.

This proposal defines a formatted-parts target for MessageFormat 2.

## Background

Past examples have shown us that if we don't provide a formatter to parts,
the string output will be re-parsed and re-processed by users.

## Use-Cases

- Markup elements
- Non-string values
- Message post-processors
- Decoration of placeholder interior parts.
  For example, identifying the separate fields in these two currency values
  (notice that the symbol, number, and fraction fields
  are not in the same order and that the separator has been omitted):
  ![image](https://github.com/unicode-org/message-format-wg/assets/69082/cb68c87f-9c0c-4bc6-b9a0-b1f97b2b789a)
  ![image](https://github.com/unicode-org/message-format-wg/assets/69082/aedd4e66-7d47-4026-8b93-4ba061bb4d84)
- Supplying bidirectional isolation of placeholders,
  such as by using HTML's `span` element with a `dir` attribute
  based on the direction of the placeholder.

## Requirements

- Define an iterable sequence of formatted part objects.
- Include metadata for each part, such as type, source, direction, and locale.
- Allow the representation of non-string values.
- Allow the representation of values that consist of an iterable sequence of formatted parts.
- Be able to represent each resolved value of a pattern with any number of formatted parts, including none.
- Define the formatted parts in a manner that allows synonymous but appropriate implementations in different programming languages.

## Constraints

- The JS Intl formatters already include formatted-parts representations for each supported data type.
  The JS implementation of the MF2 formatted-parts representation should be able to match their structure,
  at least as far as that's possible and appropriate.

## Proposed Design

The formatted-parts API is included in the spec as an optional but recommended formatting target.

The shape of the formatted-parts output is defined in a manner similar to the data model,
which includes TypeScript, JSON Schema, and XML DTD definitions of the same data structure.

At the top level, the formatted-parts result is an iterable sequence of parts.
Parts corresponding to each _text_ can be simpler than those of _expressions_,
as they do not have a `source` other than their `value`,
or set any of the other possible metadata fields.

```ts
type MessageParts = Iterable<
  MessageTextPart | MessageExpressionPart | MessageBiDiIsolationPart
>;

interface MessageTextPart {
  type: "text";
  value: string;
}
```

For MessageExpressionPart, the `source` corresponds to the expression's fallback value.
The `dir` and `locale` attributes of a part may be inherited from the message
or from the operand (if present),
or overridden by an expression attribute or formatting function,
or otherwise set by the implementation.
Each part should have at most one of `value` or `parts` defined;
some may have none.

```ts
interface MessageExpressionPart {
  type: string;
  source: string;
  parts?: Iterable<{ type: string; value: unknown }>;
  value?: unknown;
  dir?: "ltr" | "rtl" | "auto";
  locale?: string;
}
```

The bidi isolation strategies included in the spec may require
the insertion of MessageBiDiIsolationParts in the formatted-parts output.

```ts
interface MessageBiDiIsolationPart {
  type: "bidiIsolation";
  value: "\u2066" | "\u2067" | "\u2068" | "\u2069"; // LRI | RLI | FSI | PDI
}
```

Some of the MessageExpressionPart instances may be further defined
without reference to the function registry.

Unannotated expressions with a _literal_ operand
are represented by MessageStringPart.
As with MessageTextPart,
the `value` of MessageStringPart is always a string.

```ts
interface MessageStringPart {
  type: "string";
  source: string;
  value: string;
  dir?: "ltr" | "rtl" | "auto";
  locale?: string;
}
```

Unannotated expressions with a _variable_ operand
whose type is not recognized by the implementation
or for which no default formatter is available
are represented by MessageUnknownPart.

```ts
interface MessageUnknownPart {
  type: "unknown";
  source: string;
  value: unknown;
}
```

When the resolution or formatting of a placeholder fails,
it is represented in the output by MessageFallbackPart.
No `value` is provided; when formatting to a string,
the part's representation would be `'{' + source + '}'`.

```ts
interface MessageFallbackPart {
  type: "fallback";
  source: string;
}
```

Formatting functions defined in the registry
Each function defined in the registry MUST define its "formatted-parts" representation.
A function can define either a unitary string `value` or a `parts` representation.
Where possible, a function SHOULD provide a `parts` representation
if its output might reasonably consist of multiple fields.
Where available, such a formatted value should itself be represented by `parts`
rather than a unitary string `value`.
These sub-parts should not need fields beyond their `type` and `value`,
and in most cases it's presumed that the sub-part `value` would be a string.

```ts
interface MessageDateTimePart {
  type: "datetime";
  source: string;
  parts: Iterable<{ type: string; value: unknown }>;
  dir?: "ltr" | "rtl" | "auto";
  locale?: string;
}

interface MessageNumberPart {
  type: "number";
  source: string;
  parts: Iterable<{ type: string; value: unknown }>;
  dir?: "ltr" | "rtl" | "auto";
  locale?: string;
}
```

## Alternatives Considered

### Not Defining a Formatted-Parts Output

Leave it to implementations.
They will each come up with something a bit different,
but each will mostly work.

They will not be interoperable, though.

### Different Parts Shapes

See issue <a href="https://github.com/unicode-org/message-format-wg/issues/41">#41</a> for details.

They can be considered as precursors of the current proposal,
into which they've developed due to evolutionary pressure.

### Annotated String Output

Format to a string, but separately define metadata or other values.

This gets really clunky for parts that are not reasonably stringifiable.
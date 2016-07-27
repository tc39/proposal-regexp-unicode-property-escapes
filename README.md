# ECMAScript proposal: Unicode property escapes in regular expressions

## Status

This proposal is in stage 1 of [the TC39 process](https://tc39.github.io/process-document/).

## Motivation

The Unicode Standard assigns various properties and property values to every symbol. For example, to get the set of symbols that are used exclusively in the Greek script, search the Unicode database for symbols whose `Script` property is set to `Greek`.

There currently is no way to access these Unicode character properties natively in ECMAScript regular expressions. This makes it painful for developers to support full Unicode in their regular expressions. They currently have two options, neither of which is ideal:

1. Use a library such as [XRegExp](https://github.com/slevithan/xregexp) to create the regular expressions at run-time:

    ```js
    const regexGreekSymbol = XRegExp('\\p{Greek}', 'A');
    regexGreekSymbol.test('π');
    // → true
    ```

    The downside of this approach is that the XRegExp library is a run-time dependency which may not be ideal for performance-sensitive applications. For usage on the web, there is an additional load-time performance penalty: `xregexp-all-min.js.gz` takes up over 35 KB of space after minifying and applying gzip compression. Whenever the Unicode Standard is updated, a new version of XRegExp must be published and end users need to update their XRegExp copy in order to use the latest available data.

2. Use a library such as [Regenerate](https://github.com/mathiasbynens/regenerate) to generate the regular expression at build time:

    ```js
    const regenerate = require('regenerate');
    const codePoints = require('unicode-9.0.0/Script/Greek/code-points.js');
    const set = regenerate(codePoints);
    set.toString();
    // → '[\u0370-\u0373\u0375-\u0377\u037A-\u037D\u037F\u0384\u0386\u0388-\u038A\u038C\u038E-\u03A1\u03A3-\u03E1\u03F0-\u03FF\u1D26-\u1D2A\u1D5D-\u1D61\u1D66-\u1D6A\u1DBF\u1F00-\u1F15\u1F18-\u1F1D\u1F20-\u1F45\u1F48-\u1F4D\u1F50-\u1F57\u1F59\u1F5B\u1F5D\u1F5F-\u1F7D\u1F80-\u1FB4\u1FB6-\u1FC4\u1FC6-\u1FD3\u1FD6-\u1FDB\u1FDD-\u1FEF\u1FF2-\u1FF4\u1FF6-\u1FFE\u2126\uAB65]|\uD800[\uDD40-\uDD8E\uDDA0]|\uD834[\uDE00-\uDE45]'
    // Imagine there’s more code here to save this pattern to a file.
    ```

    This approach results in optimal run-time performance, although the generated regular expressions tend to be fairly large in size (which could lead to load-time performance problems on the web). The biggest downside is that it requires a build script, which gets painful as the developer needs more Unicode-aware regular expressions. Whenever the Unicode Standard is updated, the build script must be updated and its results must be deployed in order to use the latest available data.

## Proposed solution

We propose the addition of _Unicode property escapes_ of the form `\p{…}` and `\P{…}`. Unicode property escapes are a new type of escape sequence available in regular expressions that have the `u` flag set. With this feature, the above regular expression could be written as:

```js
const regexGreekSymbol = /\p{Script=Greek}/u;
regexGreekSymbol.test('π');
// → true
```

This proposal solves all the abovementioned problems:

* It is no longer painful to create Unicode-aware regular expressions.
* There is no dependency on run-time libraries.
* The regular expressions patterns are compact and readable — no more file size bloat.
* Creating a script that generates the regular expression at build time is no longer necessary.
* Code that uses Unicode property escapes stays up-to-date “automatically” from the developer’s point of view: whenever the Unicode Standard gets an update, the ECMAScript engine updates its data.

## High-level API

Unicode property escapes for non-binary Unicode properties look like this:

<pre>\p{<b><i>UnicodePropertyName</i></b>=<b><i>UnicodePropertyValue</i></b>}</pre>

The aliases defined in [`PropertyAliases.txt`](http://unicode.org/Public/UNIDATA/PropertyAliases.txt) and [`PropertyValueAliases.txt`](http://unicode.org/Public/UNIDATA/PropertyValueAliases.txt) may be used instead of the canonical property and value names. The use of an unknown property name or value triggers a `SyntaxError`.

For binary properties, the following syntax is available:

<pre>\p{<b><i>LoneUnicodePropertyNameOrValue</i></b>}</pre>

This syntax may also be used as a shorthand for `General_Category` values, e.g. `\p{Letter}` instead of `\p{General_Category=Letter}`.

`\P{…}` is the negated form of `\p{…}`.

Implementations must support the following Unicode properties and their property aliases as required by [UTS18 RL1.2](http://unicode.org/reports/tr18/#RL1.2): `General_Category`, `Script`, `Script_Extensions`, `Alphabetic`, `Uppercase`, `Lowercase`, `White_Space`, `Noncharacter_Code_Point`, `Default_Ignorable_Code_Point`, `Any`, `ASCII`, and `Assigned`. Implementations may extend Unicode property support to the remaining enumeration or binary properties.

### FAQ

#### What about backwards compatibility?

In regular expressions without the `u` flag, the pattern `\p` is an (unnecessary) escape sequence for `p`. Patterns of the form `\p{Letter}` might already be present in existing regular expressions without the `u` flag, and therefore we cannot assign new meaning to such patterns without breaking backwards compatibility.

For this reason, ECMAScript 2015 made unnecessary escape sequences like `\p` and `\P` [throw an exception](https://bugs.ecmascript.org/show_bug.cgi?id=3157) when the `u` flag is set. This enables us to change the meaning of `\p{…}` and `\P{…}` in regular expressions with the `u` flag without breaking backwards compatibility.

#### Why not support loose matching?

[UAX44-LM3](http://unicode.org/reports/tr44/#Matching_Symbolic) specifies the loose matching rules for comparing Unicode property and value aliases.

> Ignore case, whitespace, underscores, hyphens, […]

Loose matching makes `\p{lB=Ba}` equivalent to `\p{Line_Break=Break_After}` or `/\p{___lower C-A-S-E___}/u` equivalent to `/\p{Lowercase}/u`. We assert that this feature does not add any value, and in fact harms code readability and maintainability.

Should the need arise, then support for loose matching can always be added later, as part of a separate ECMAScript proposal. If we add it now, however, there is no going back.

#### Why not support the `is` prefix?

[UAX44-LM3](http://unicode.org/reports/tr44/#Matching_Symbolic) specifies the loose matching rules for comparing Unicode property and value aliases, one of which is:

> Ignore […] any initial prefix string `is`.

This rule makes `Script=IsGreek` and `IsScript=Greek` equivalent to `Script=Greek`. We assert that this feature does not add any value, and in fact harms code readability. It introduces ambiguity and increases implementation complexity, since some property values or aliases already start with `is`, e.g. `Decomposition_Type=Isolated` and `Line_Break=IS` which is an alias for `Line_Break=Infix_Numeric`.

Compatibility with Unicode property escapes in other languages is not an argument either, since [no existing regular expression engine](http://unicode.org/mail-arch/unicode-ml/y2016-m06/0012.html) seems to implement the `is` prefix exactly as described in UAX44-LM3, and those that partially implement it wildly differ in behavior.

Strictness is preferred over ambiguity.

Should the need arise, then support for the `is` prefix can always be added later, as part of a separate ECMAScript proposal. If we add it now, however, there is no going back.

#### Why not support e.g. `\pL` as a shorthand for `\p{L}`?

This shorthand doesn’t add any value and as such the added implementation complexity (small as it may be) isn’t worth it. `\p{L}` works; there’s no reason to introduce another syntax for it other than compatibility with other languages which is an utopian goal anyhow.

Should the need arise, then support for this shorthand can always be added later, as part of a separate ECMAScript proposal. If we add it now, however, there is no going back.

#### Why not support `:` as a separator in addition to `=`?

Supporting multiple separators doesn’t add any value and as such the added implementation complexity (small as it may be) isn’t worth it. `\p{Block=Arrows}` works; there’s no reason to introduce another syntax for it other than compatibility with other languages which is an utopian goal anyhow.

Should the need arise, then support for the `:` separator can always be added later, as part of a separate ECMAScript proposal. If we add it now, however, there is no going back.

#### Why not support e.g. `\p{ScriptName}` as a shorthand for `\p{Script=ScriptName}`?

In the majority of use cases, `Script_Extensions` should be used over `Script`. [UTS24](http://unicode.org/reports/tr24/#Multiple_Script_Values) explains this nicely with practical examples. As such, it would make more sense to add a shorthand for `Script_Extensions` than for `Script`. Doing either would cause confusion, however, since the sets of values for these two properties are identical. For example, it wouldn’t be clear if `\p{Old_Persian}` refers to the `Script` or `Script_Extensions` with that name.

## Illustrative examples

### Unicode-aware version of `\d`

To match any decimal number in Unicode rather than just ASCII `[0-9]`, use `\p{Decimal_Number}` instead of `\d` as per [UTS18](http://unicode.org/reports/tr18/#digit).

```js
const regex = /^\p{Decimal_Number}+$/u;
regex.test('𝟏𝟐𝟑𝟜𝟝𝟞𝟩𝟪𝟫𝟬𝟭𝟮𝟯𝟺𝟻𝟼');
// → true
```

### Unicode-aware version of `\D`

To match any Unicode symbol that is not a decimal number rather than just `[^0-9]`, use `\P{Decimal_Number}` instead of `\D`.

```js
const regex = /^\P{Decimal_Number}+$/u;
regex.test('Իմ օդաթիռը լի է օձաձկերով');
// → true
```

### Unicode-aware version of `\w`

To match any word symbol in Unicode rather than just ASCII `[a-zA-Z0-9_]`, use `[\p{Alphabetic}\p{Mark}\p{Decimal_Number}\p{Connector_Punctuation}\p{Join_Control}]` as per [UTS18](http://unicode.org/reports/tr18/#word).

```js
const regex = /([\p{Alphabetic}\p{Mark}\p{Decimal_Number}\p{Connector_Punctuation}\p{Join_Control}]+)/gu;
const text = `
Amharic: የኔ ማንዣበቢያ መኪና በዓሣዎች ተሞልቷል
Bengali: আমার হভারক্রাফ্ট কুঁচে মাছ-এ ভরা হয়ে গেছে
Georgian: ჩემი ხომალდი საჰაერო ბალიშზე სავსეა გველთევზებით
Macedonian: Моето летачко возило е полно со јагули
Vietnamese: Tàu cánh ngầm của tôi đầy lươn
`;

let match;
while (match = regex.exec(text)) {
  const word = match[1];
  console.log(`Matched word with length ${ word.length }: ${ word }`);
}
```

Console output:

```
Matched word with length 7: Amharic
Matched word with length 2: የኔ
Matched word with length 6: ማንዣበቢያ
Matched word with length 3: መኪና
Matched word with length 5: በዓሣዎች
Matched word with length 5: ተሞልቷል
Matched word with length 7: Bengali
Matched word with length 4: আমার
Matched word with length 11: হভারক্রাফ্ট
Matched word with length 5: কুঁচে
Matched word with length 3: মাছ
Matched word with length 1: এ
Matched word with length 3: ভরা
Matched word with length 3: হয়ে
Matched word with length 4: গেছে
Matched word with length 8: Georgian
Matched word with length 4: ჩემი
Matched word with length 7: ხომალდი
Matched word with length 7: საჰაერო
Matched word with length 7: ბალიშზე
Matched word with length 6: სავსეა
Matched word with length 12: გველთევზებით
Matched word with length 10: Macedonian
Matched word with length 5: Моето
Matched word with length 7: летачко
Matched word with length 6: возило
Matched word with length 1: е
Matched word with length 5: полно
Matched word with length 2: со
Matched word with length 6: јагули
Matched word with length 10: Vietnamese
Matched word with length 3: Tàu
Matched word with length 4: cánh
Matched word with length 4: ngầm
Matched word with length 3: của
Matched word with length 3: tôi
Matched word with length 3: đầy
Matched word with length 4: lươn
```

### Unicode-aware version of `\W`

To match any non-word symbol in Unicode rather than just `[^a-zA-Z0-9_]`, use `[^\p{Alphabetic}\p{Mark}\p{Decimal_Number}\p{Connector_Punctuation}\p{Join_Control}]`.

### Other examples

Match any numeric symbol in Unicode, including non-decimal symbols such as Roman numerals:

```js
const regex = /^\p{Number}+$/u;
regex.test('²³¹¼½¾𝟏𝟐𝟑𝟜𝟝𝟞𝟩𝟪𝟫𝟬𝟭𝟮𝟯𝟺𝟻𝟼㉛㉜㉝ⅠⅡⅢⅣⅤⅥⅦⅧⅨⅩⅪⅫⅬⅭⅮⅯⅰⅱⅲⅳⅴⅵⅶⅷⅸⅹⅺⅻⅼⅽⅾⅿ');
// → true
```

Match any symbol in the `Arrows` block:

```js
const regexArrows = /^\p{Block=Arrows}+$/u;
regexArrows.test('←↑→↓↔↕↖↗↘↙↚↛↜↝↞↟↠↡↢↣↤↥↦↧↨↩↪↫↬↭↮↯↰↱↲↳↴↵↶↷↸↹↺↻↼↽↾↿⇀⇁⇂⇃⇄⇅⇆⇇⇈⇉⇊⇋⇌⇍⇎⇏⇐⇑⇒⇓⇔⇕⇖⇗⇘⇙⇚⇛⇜⇝⇞⇟⇠⇡⇢⇣⇤⇥⇦⇧⇨⇩⇪⇫⇬⇭⇮⇯⇰⇱⇲⇳⇴⇵⇶⇷⇸⇹⇺⇻⇼⇽⇾⇿');
// → true
```

Match ECMAScript [`IdentifierStart`](https://tc39.github.io/ecma262/#prod-IdentifierStart) or [`IdentifierPart`](https://tc39.github.io/ecma262/#prod-IdentifierPart) symbols [without the need for complex regular expressions generated by build scripts](https://gist.github.com/mathiasbynens/6334847):

```js
const regexIdentifierStart = /[$_\p{ID_Start}]/u;
const regexIdentifierPart = /[$_\u200C\u200D\p{ID_Continue}\p{Other_ID_Start}]/u;
// Note: the following doesn’t account for reserved words in order to
// keep the example simple.
const regexIdentifier = /^(?:[$_\p{ID_Start}])(?:[$_\u200C\u200D\p{ID_Continue}\p{Other_ID_Start}])*$/u;
```

## Specification

* [Ecmarkup source](https://github.com/mathiasbynens/es-regexp-unicode-property-escapes/blob/master/spec.html)
* [HTML version](https://mathiasbynens.github.io/es-regexp-unicode-property-escapes/)

## Implementations

* [V8](https://bugs.chromium.org/p/v8/issues/detail?id=4743) with the `--harmony_regexp_property` flag set
* [regexpu (transpiler)](https://github.com/mathiasbynens/regexpu) with the `{ unicodePropertyEscape: true }` option enabled
    * [online demo](https://mothereff.in/regexpu#input=/%5Cp%7BLetter%7D/u&unicodePropertyEscape=1)
    * [exhaustive list of supported properties](https://github.com/mathiasbynens/regexpu-core/blob/master/property-escapes.md)
    * [Babel plugin](https://github.com/mathiasbynens/babel-plugin-transform-unicode-property-regex)

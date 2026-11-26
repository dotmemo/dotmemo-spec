# The dotmemo format specification, grammar `dotmemo/1`

- Status: draft
- Author: Aurèle ZANNOU TCHOKO
- Version: 0.1.0
- Date: 2026-11-20
- License: [CC-BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/)
- Feedback: [https://github.com/dotmemo/dotmemo-spec/issues/new](https://github.com/dotmemo/dotmemo-spec/issues/new)

## Summary

A dotmemo file (`.dotmemo`) is plain UTF-8 text: notes, lists, amounts,
currencies, dates, times, contacts, links, and calculations, written the
way people write on paper.
This document is the exchange contract for that text. It brings together:

- what a reader must preserve (#4);
- the grammar: how the text splits into sections, blocks, lines, and items, and
  the forms that are recognized (#5 to #13);
- the calculation rules (#14);
- the reference lexicon (#16);
- the conformance levels and cases (#18, appendices A to C).

Two names recur: `.dotmemo` is the file extension, `dotmemo/1` the version of
the grammar that reads it. A file may write that version on its first line, with
its settings (#6.6). The same `.dotmemo` file will one day be readable by a
`dotmemo/2` grammar; the extension itself will not change.

## Contents

1. Scope
2. Principles
3. Conventions
4. What the reader must preserve
5. Encoding and characters
6. Structure
7. Inline lexicon
8. Compound forms
9. Variables
10. Groups and totals
11. Calculation blocks
12. Taxes
13. Fixed-point entry
14. Evaluation
15. Decisions outside the format
16. Reference lexicon
17. Reserved symbols and versions
18. Conformance
19. Security and privacy
20. Complete example
21. Open questions

- Appendix A. Syntax tree in JSON
- Appendix B. Evaluation results in JSON
- Appendix C. Conformance cases

## 1. Scope

This document defines the **public exchange format** `dotmemo/1`:

- the text of a `.dotmemo` file and what must be preserved of it;
- the deterministic syntax: which forms are recognized, and where;
- the deterministic calculation of values: variables, expressions, taxes, calculation blocks,
  counted amounts;
- the settings and outside decisions those rules depend on, as inputs;
- two verification JSON formats (appendices A and B) and the cases that fix the expected behavior
  (appendix C).

It does not define:

- the **probabilistic interpretation**: deciding that a number is an amount, its nature (expense,
  income…), its category, dates in words (`tomorrow`, `monday`, `dimanche`, `hier`, etc.), unmarked
  persons, unmarked tasks. An implementation may have one or not (#15);
- the storage, history, encryption, synchronization, or archiving of notebooks;
- the display.

## 2. Principles

1. **Every input is valid.** No text produces a syntax error. Whatever is not recognized is text.
   _The user's text is sovereign._ A reader or evaluator never modifies the source file to record
   its results. Computed values are a derived projection (typically shown in the margin), never a
   rewrite of the text.
2. **Lossless.** For any text `t`: `text(tree(t)) == t`, byte for byte.
3. **Language neutrality.** The grammar uses symbols only. The words (currencies, keywords,
   multipliers, functions) come from a lexicon (#16). The French and English lexicons are active
   together, whatever the regional setting.
4. **Nothing is mandatory.** Symbols serve to resolve an ambiguity or to force an interpretation; a
   text with no symbol at all is still valid text. An ambiguity or an unrecognized form must never
   block the reading of the rest of the document.
5. **Determinism.** The same text, the same format version, the same lexicon, and the same settings
   always give the same tree and the same values, on every machine.

## 3. Conventions

### 3.1 Normative vocabulary

- **MUST**: required. **MUST NOT**: forbidden.
- **SHOULD**: recommended; departing from it needs a reason.
- **MAY**: optional.

Every normative rule in this document is deterministic. The appendix C cases are part of it: where
the text and a case disagree, the disagreement is a defect of the document, to be reported (#21);
until it is fixed, the case prevails.

### 3.2 Roles

- **Reader**: a program that reads and rewrites `.dotmemo` files (an editor, a conversion tool, a
  file sync). It need not understand the text.
- **Parser**: a program that produces the syntax tree of a text (#5 to #13).
- **Evaluator**: a program that computes the values from the tree (#14).

### 3.3 Terms

- **Notebook**: the whole of one person's or one team's notes; a `.dotmemo` file holds all or part of
  it.
- **Settings**: parameters of the notebook that are not in the text: regional setting, currency,
  taxes, fixed-point entry (#14.1).
- **Item**: a piece of a line's content recognized by the parser (number, amount, mention…).
- The forms `YYYY-MM-DD` denote a four-digit year, a two-digit month, and a two-digit day.
- In the examples, `/` separates successive lines of one block: `100` / `- 20` / `=`.

## 4. What the reader must preserve

The text is the only source of truth. Everything a program shows or computes derives from the text
and the settings; nothing computed is written into the text.

### 4.1 On import

A reader that receives text from an outside source (a file, the clipboard) MUST apply, in this
order, these transformations only:

1. remove an initial UTF-8 BOM (`EF BB BF`);
2. convert each CRLF, then each lone CR, into LF.

A text that contains an invalid UTF-8 byte sequence is not a `dotmemo/1` text: the import MUST
refuse it. A program MAY offer to convert it from another encoding, but that is then a conversion,
which the person sees, and not a lossless import.

### 4.2 After import

The resulting text is the **stored text**. A reader MUST keep it and rewrite it byte for byte. In
particular, it MUST NOT:

- normalize Unicode (NFC, NFD, NFKC…): a precomposed `é` and an `e` followed by U+0301 stay
  distinct in the text, even though they compare equal (#5);
- remove or add trailing whitespace, convert tabs, change indentation;
- merge, remove, or add blank lines;
- add or remove the final line break: the presence or absence of a LF at the end of the file is
  part of the text;
- replace invisible whitespace (U+00A0, U+202F) with spaces;
- remove escapes (`\:` stays `\:`);
- rewrite a separator line (#6.2), including its free text;
- convert non-ASCII digits to ASCII digits;
- reorder lines, blocks, or sections;
- write a computed value (a total, a result, a tax, a rounded amount) into the text.

A reader that does not recognize a form MUST keep it as it is: it is text. The reserved symbols
(#17) are text in `dotmemo/1` and MUST be preserved.

### 4.3 Verifiable property

For any stored text `t`, a conforming parser produces a tree whose rewrite gives `t` back: the text
of each line is the concatenation of its prefix and the text of its items, and the file is the join
of the lines by LF, followed by a LF if the file ends with one (appendix A).

## 5. Encoding and characters

- The text is **UTF-8**, line endings are **LF** (U+000A) (#4.1).
- **Comparisons**: the stored text is never normalized (#4.2); only comparisons are.
  - Identifiers and lexicon words: NFC, then full Unicode case folding by default, without the
    Turkish variant.
  - Persons and tags: full case folding, then NFD decomposition, removal of characters of category
    `M`, then NFC recomposition. `Payé` and `paye` are brought together.
- **Unicode versions**: the general categories and case folding follow Unicode 16.0; the NFC and
  NFD normalizations follow Unicode 18.0. Another version may change a tree on recent characters: an
  implementation MUST declare the versions it uses.
- **Whitespace**: U+0020 (space), U+0009 (tab), U+00A0 (no-break space), U+202F (narrow no-break
  space, common before `:` and `%` in French typography).
- **Blank line**: an empty line or one made only of whitespace.
- **Letter**: a character of Unicode category `L`. **Mark**: category `M` (for example a combining
  accent U+0301 in unnormalized text). **Digit**: `0` to `9` (ASCII) only. Other Unicode digits
  (`٣`, `３`) never start a number, a date, a numbered bullet, or a separator line.
- **Case**: ISO 4217 currency codes (`USD`, `EUR`, `XOF`) and single-letter multipliers (`k`, `K`,
  `M`) compare as written, case included. Other lexicon words compare after case folding: `DOLLAR`,
  `dollar`, and `Dollar` are equivalent, `ROUND`, `Round` and `round` are equivalent, but `USD` is not `Usd` and `EUR` is not `eur`.

## 6. Structure

### 6.1 Hierarchy

```text
file     = header? + notebook
notebook = section*
section  = separator line? + block*
block    = a non-blank line, followed by consecutive non-blank lines
line     = prefix + content
```

One or more blank lines separate two blocks; they are kept as they are. A separator line belongs to
no block: it also ends the block before it.

**Splitting into lines.** Empty text has no line. Otherwise, split on each LF keeping the empty
pieces; if the text ends with a LF, drop the last piece, which is empty, and note that final LF.
So `"\n"` is one empty line followed by a final LF, and `"a"` one line with no final LF. Lines are
numbered from 1, blank, and separator lines included.

### 6.2 Day sections

A section begins with a **separator line**:

```text
=== 2026-10-29 ===
=== 2026-10-29 Thursday ===
=== 2026-10-29 jeudi ===
```

Form: optional whitespace, at least three `=`, at least one whitespace, a valid date `YYYY-MM-DD`
(#8.6), optionally a whitespace followed by free text that is ignored, at least one whitespace, at
least three `=`, optional whitespace.

- The text placed before the first separator belongs to the day of import.
- A line that looks like a separator but whose date is invalid is text: `=== 2026-02-30 ===`,
  `=== 0000-01-01 ===`, and `=== ٢٠٢٦-١٠-٢٩ ===` are not separators.

### 6.3 Indentation

Indentation is optional.

- **Width** of a line: the number of columns of leading whitespace. A tab counts as 4 columns; any
  other whitespace counts as 1.
- The **parent** of a line is the nearest preceding line, in the same block, whose width is
  strictly smaller. A line with no parent is at the root level of the block.
- No consistency of unit is required: 2, 3, or 4 spaces, or tabs, mix freely.

### 6.4 Titles and groups

- A line is a **title** if its content, with trailing whitespace removed, ends with an unescaped
  `:` (U+003A). Whitespace before the `:` is allowed (`groceries :`).
- The **label** of the title is its content before that `:`, with trailing whitespace removed.
- The **items** of a title of width `w` are determined from the line that follows it in the block:
  - if it is more indented than the title, the group is **indented**: its items are the following
    lines of width strictly greater than `w`, up to the first line of width less than or equal to
    `w`;
  - if it has the same width, the group is **flat**: its items are the following lines, up to the
    first of these cases: the end of the block; a line of width strictly less than `w`; a line of
    width `w` that is itself a title;
  - otherwise (a title at the end of a block, or a following line that is less indented), the group
    is empty.
- A title placed among the items of a group opens a **subgroup**; it is itself an item of the group
  that contains it.
- Two titles of the same width in a flat group are **neighbouring groups**: the second closes the
  first.
- The **group** of a line is the innermost of the groups it is an item of.
- To keep a line that ends with `:` as text, write `\:`.

Flat groups:

```text
First proposal:
quoted price 2400
fees 600

Second proposal:
quoted price 2700
fees 500
-10% discount
```

Each text gives two neighbouring groups: the first has 2 items, the second 3.

Indented group:

```text
Second proposal:
    quoted price 2700
    fees 500
    -10% discount
    meeting:
        at 5 PM
        Conference room R2
        confirmed by email
    priority B
    [ ] write the report before Tuesday
```

`meeting` is an indented group that stops before `priority B`; `priority B` is an item of
`Second proposal`, not of `meeting`.

### 6.5 Line prefixes

After the indentation, in this order, each optional:

1. **Bullet**: `-`, `*`, `•` (U+2022), or `–` (U+2013), followed by at least one whitespace.
   **Numbered bullet**: 1 to 3 digits followed by `.` or `)`, then at least one whitespace. The
   number is never an amount. `+` is never a bullet: it marks money in (#8.3). In a calculation
   block, a line that begins with `-` or `*`, followed by a whitespace then a value, follows the
   rule of #11.3: bullet or operation.
2. **Task box**: `[ ]` (to do, with a U+0020 space), `[x]` or `[X]` (done), followed by a whitespace
   or the end of line.

The **prefix** of a line is its indentation, its bullet, and its box, with the whitespace that
follows them. The **content** is the rest of the line.

### 6.6 File header

A file may begin with a **header** that gives its grammar version and the settings it reads with. A
file (with a header) thus gives the same trees and the same totals to every reader.

```text
!dotmemo/1
!region en-US
!currency USD
!tax sales_tax 8.875 % 2024-01-01
!fixed-point no
```

```text
!dotmemo/1
!région fr-SN
!devise XOF
!taxe tva 18 % 2024-01-01
!virgule-fixe non
```

- The **version line** is line 1: `!dotmemo/`, a number with no leading zero, then optional
  whitespace. `!dotmemo/1 draft`, `!dotmemo/01` and ` !dotmemo/1` are not version lines.

- The header is the version line and the following lines that start with `!`, in the first column,
  up to the first that does not start with `!`. Its lines belong to no block.

- A **setting line** is `!`, a key, then its values, separated by whitespace. The key compares
  after case folding and removal of diacritics (#5):

  | Key | Values | Setting |
  |---|---|---|
  | `region`, `région` | a label `en-XX` or `fr-XX` (#16.1) | regional setting |
  | `currency`, `devise` | a code of three ASCII uppercase letters | currency |
  | `tax`, `taxe` | an identifier (#8.10); a rate in percent: digits, then at most one `.` or `,` followed by digits; `%`, attached or separate, optional; an optional date `YYYY-MM-DD` | a rate of this tax, in force at that date, or from always |
  | `fixed-point`, `virgule-fixe` | `yes`, `no`, `oui`, `non`, case-insensitive | fixed-point entry |
  | `day`, `jour` | a date `YYYY-MM-DD` | the day of the lines placed before the first separator line |

- An unknown key, or values that do not follow the table, leave the line in the header with no
  effect. A future version can thus add keys without disturbing a reader of today.

- A version line that names another version (`!dotmemo/2`) still opens a header; its known settings
  apply, and a reader SHOULD say that the file comes from a more recent version.

- **Effect.** The header settings replace the reader's for this file: the regional setting for
  parsing; the currency, the fixed point, and the rates for the calculation; the day for the
  effective date (#15.1). As soon as the header names a tax, its taxes replace all others. The
  block formats stay the reader's.

- Without an exact version line, there is no header: lines that start with `!` are text (#17).

- A program that **imports** a file into a set of notes that already has its settings MAY keep its
  own; it SHOULD then say which ones differ, and read the file's numbers with the header's regional
  setting.

- A program that **writes** a file SHOULD put a complete header in it.

## 7. Inline lexicon

The content of a line is split left to right into simple items. At each position, the classes below
are tried in order; the longest match of the first class that succeeds is kept. Every position is
covered by at least one class.

| Order | Class | Form |
|---|---|---|
| 1 | escape | `\` followed by an ASCII punctuation character: that character becomes text; a `\` followed by anything else is a literal `\` |
| 2 | url | #8.13 |
| 3 | email | #8.14 |
| 4 | mention | #8.11 |
| 5 | tag | #8.12 |
| 6 | digits | a run of digits `0` to `9` |
| 7 | currency symbol | the lexicon's symbols (#16.6) |
| 8 | operator | a character among `+ - − * × / ^ ² ³ ( ) % = ;` |
| 9 | word | a letter, then letters, marks, digits, or `_`; right after digits, letters and marks only |
| 10 | punctuation | any other non-blank character, one per item |
| 11 | whitespace | a run of whitespace |

- Apostrophes and hyphens are not part of words: `well-known` gives `well`, `-`, `known`; `O'Brien`
  gives `O`, `'`, `Brien`. Free text is not affected by this; it lets you write `price-discount`
  between two variables (#8.8).
- A word attached to digits stops at the first digit: `3x1000` gives `3`, `x`, `1000`; `15h30`
  gives `15`, `h`, `30`.

The lexicon decides no meaning. The compound forms (#8) are then recognized over the sequence of
items, left to right. At each position, these are tried in order: date (#8.6), time (#8.5), phone
(#8.7), expression (#8.8), amount (#8.3), percentage (#8.4), number (#8.1), variable reference (a
word that names a visible variable, #9). The first form that succeeds is kept, with its longest
match. Whatever belongs to no form is text.

In a **calculation context** (#8.8), dates, times, and phones are not tried.

## 8. Compound forms

### 8.1 Numbers

- A number begins with a digit: one or more digits, then optional thousands groups, then an
  optional decimal part.
- **Thousands separators**, by the language of the regional setting (#16.1): in `en`, `,` or a
  whitespace; in `fr`, a whitespace or `.`. A thousands group is a single separator followed by
  exactly 3 digits (not followed by another digit). Thousands groups are admitted only if the first
  group has 1 to 3 digits, and all the thousands separators of one number are identical (the four
  whitespaces count as a single separator).
- **Greedy reading**: groups are consumed as long as they are valid. `1 500 2 000` gives two
  numbers (`1 500` and `2 000`), `1 500 2000` gives `1 500` and `2000`, and `15000 000` gives
  `15000` and `000`. In `en`, `15000,000` gives `15000`, `,`, `000`.
- **Decimal separator**: `.` in `en`, `,` in `fr`, followed by at least one digit. A trailing
  separator is not part of the number: in `en`, `1500.` is the number `1500` followed by `.`; in
  `fr`, `1500,` is `1500` followed by `,`.
- A separator that forms neither a valid thousands group nor a decimal part ends the number and
  stays outside. In `en`, `1,50` gives `1`, `,`, `50`, and `1.2.3` gives `1.2`, `.`, `3`; in `fr`,
  `1.50` gives `1`, `.`, `50`.
  - **Ambiguous case**: a number with 1 to 3 leading digits and a single separator (`,` or `.`)
  followed by exactly 3 digits (`1,500` or `1.500`). Its value follows the regional setting (`1,500`
  is `1500` in `en` and `1.5` in `fr`); the number is marked **ambiguous**. Evaluation may
  reinterpret it according to the currency (#14.10).
- **Block format**: a block may declare its own regional setting, which replaces the notebook's for
  that block (setting `block_formats`, #14.1). This is useful for text pasted from a source in
  another region.
- **In a function call** (#8.8), in `en`, `,` separates the arguments and is never a thousands
  separator.
- A number of 8 to 15 contiguous digits, with no separator, is marked a **phone candidate** (#8.7).

### 8.2 Multipliers

- A multiplier is a whole word (#7), attached to the number or separated from it by a single
  whitespace: `2k`, `1.5 M`, `2 billion` (in `en`), `1,5 M`, `2 millions` (in `fr`). A number takes
  at most one multiplier; it is part of the number and multiplies its value. `2kg` has no multiplier
  (the whole word is `kg`). List and case: #16.5.
- Lowercase `m` is not a multiplier (meter). A visible variable that bears a multiplier's name
  masks it (#9, rule 5).
- **Initials and courtesy titles**: a single-letter word followed, attached, by a `.` then a word
  (#7), attached or after whitespace, is an initial or a courtesy title, not a multiplier:
  `77 123 45 67 M. Sam`, `2 k. Sam`, `67M. Sam`. It stays a multiplier at the end of a line or
  before something other than a word (the `.` stays outside the number): `sale 67 M.`,
  `vente 67 M.`, `67 M. 12`. Multi-letter multipliers are not concerned: `2 Md. Sam` is `2 × 10⁹`.

### 8.3 Sign and explicit amount

- **Sign**: `+`, `-`, or `−` (U+2212) attached to a number, or to a currency followed by a number,
  when no operand precedes it: start of content, text, `(`, or an operator. Otherwise it is a
  subtraction: `1500 -500` is 1000 (#8.8).
- At the start of content, `+` followed by whitespace then a number is also a sign
  (`+ 5000 salary`). `- 500` (with a whitespace) at the start of a line is a bullet followed by
  500, except in a calculation block (#11.3).
- **Explicit amount**: `[sign] [currency] number [currency]`, with a single currency, attached to
  the number or separated from it by a single whitespace. The currency is a symbol, an ISO 4217
  code, or a lexicon name (`dollars`, `euros`). A signed number with no currency is also an explicit
  amount (`+5000`).
- `F`, `FCFA`, and `CFA` denote the CFA franc of the active regional setting: `XAF` if the regional
  setting's country is in the CEMAC zone (#16.3), `XOF` otherwise.
- A single-letter currency word followed, attached, by a `.` then a word is an initial, as for
  multipliers (#8.2): `2500 F. Sam` is the number `2500`, `77 123 45 67 F. Sam` a phone candidate.
  `sale 2500 F.` and `2500 F. 12` stay amounts.

### 8.4 Percentage

A number followed by `%`, attached or separated by a single whitespace: `19%`, `19 %` (with a
narrow no-break space in French typography).

### 8.5 Time

A time is valid if the hours run from 0 to 23 and the minutes from 0 to 59 (from 1 to 12 with
`am`/`pm`). An invalid form is not a time.

Exact forms, with `B` a single optional whitespace:

| Form | Examples |
|---|---|
| `H` `:` `MM` [`B` `am`/`pm`] | `15:30`, `3:30pm`, `3:30 PM` |
| `H` `B` `am`/`pm` | `3pm`, `3 pm` |
| `H` `h` [`MM`] | `9h`, `15h30`, `15H30` |
| `H` whitespace `h` [whitespace `MM`] | `15 h`, `15 h 30` |

- `12am` is 00:00 and `12pm` is 12:00.
- `H`: 1 or 2 digits. `MM`: exactly 2 digits. `am` and `pm` in uppercase or lowercase.
- If digits follow `h` or `:`, they must form valid minutes, otherwise the form is not a time:
  `15h75` and `15h5` are not times.

### 8.6 Numeric date

Forms: `YYYY-MM-DD`, `DD/MM`, `DD/MM/YY`, `DD/MM/YYYY`, `DD-MM-YYYY`, `DD.MM.YYYY`.

- **Day/month order**: it follows the regional setting (#16.2). In a region with month-day order
  (like `en-US`), every form other than `YYYY-MM-DD` reads month then day:
  - `03/11/2026` denotes 11 March 2026 in `en-US`, and 3 November 2026 in `en-GB` and in `fr`.
  - `05/13/2026` is valid in `en-US`, but not in `en-GB` or `fr`; `13/05/2026` is valid in `en-GB`
    and `fr`, but not in `en-US`.
- **Field widths**: `YYYY` has 4 digits; `YY` has 2 digits, read `20YY` (`00` is `2000`); `MM` and
  `DD` have exactly 2 digits in the `YYYY-MM-DD` form, and 1 or 2 digits in the other forms
  (`2026-3-5` is not a date, `3/5/2026` is one).
- **Boundaries and no partial fallback**: a numeric date cannot be immediately preceded or followed
  by a digit, nor extended by the same separator attached to a digit. A longer sequence or an
  invalid complete date never falls back to a shorter form:
  - `1/2/3`, `12/03/2026/4`, and `2026-03-12-01` produce no date.
  - `29/02/2026` (invalid) does not produce the date candidate `29/02`.
- **Calendar validity**: the date must exist in the proleptic Gregorian calendar, from year 1 to
  year 9999, otherwise it is not a date:
  - **Valid**: `2024-02-29`, `02/29/24` (in `en-US`), `29/02/00` (in `en-GB` and `fr`).
  - **Invalid**: `2026-02-30`, `2026-02-29`, `01/01/0000`.
- **Date candidate (`DD/MM` or `MM/DD`)**: the form without a year is a **date candidate**; it is
  enough that the date exists in at least one year (`2/29` in `en-US` and `29/02` in `en-GB`/`fr`
  are candidates). `meeting 3/11` or `rdv 3/11` may be a date, while `1/2 lb` or `1/2 kg` is rather
  a fraction: the choice belongs to interpretation (#15).
- In a **calculation context** (#8.8), no numeric date is recognized: `x = 1/2` and `x = 12/04/20`
  are divisions, `x = 2026-10-05` is a subtraction.
- **Dates in words**: expressions like `tomorrow`, `Oct 28`, `next Monday`, `hier`, `lundi`, or
  `28 oct` are not numeric dates; they belong to interpretation (#15).

### 8.7 Phone

1. **Phone**: `+`, with no operand before it, attached to groups of digits each separated by a
   single whitespace, dot, or hyphen, totalling 8 to 15 digits (E.164 form): `+1 555 123 4567`,
   `+33612345678`, `+221 77 123 45 67`. If there are several groups, the first has 1 to 3 digits,
   those of a country code: `+5000 3000` is an amount followed by a number.

2. **Phone candidate**: at least two groups of 2 to 4 digits, all separated by the same separator, a
   single whitespace, dot, or hyphen (the four whitespaces count as one), totalling 8 to 15 digits,
   when the whole sequence is not a valid number (#8.1): `77 123 45 67`, `77.123.45.67`.
   `771 234 567` is a number. A group of 4 digits is admitted only if:
   - the first group begins with `0`, like a national prefix: `0612 345 678`, `06 1234 5678`;
   - or the sequence, written with `.` or `-`, has the form 3-3-4 (`555-123-4567`), 4-2-2, or 2-2-4
     (`2026-13-45`, a date that does not exist and does not become a subtraction).

   Otherwise these groups are numbers, side by side or in an expression: `1500 2000`, `2500 12 25`,
   `5000-2500`, `1500-500-200`.

3. **Phone-candidate number**: 8 to 15 contiguous digits form a number marked a phone candidate
   (#8.1): `0612345678`, but also `15000000`.

Forms 2 and 3 are candidates: interpretation decides (#15). A reader without interpretation treats
form 3 as a number and form 2 by the ordinary rules of numbers and operators.

A sequence of groups (forms 1 and 2) starts no other form. The longest sequence that respects all
the rules is kept, and none if no prefix respects them:

- **Left boundary (form 2)**: the sequence must not be preceded, attached, by a letter, a digit, or
  a `.` or `-` preceded by a digit, nor begin a prefixed-currency amount (#8.3): `ref555-123-4567`
  is not a candidate, `$555-123-4567` begins with the amount `$555`.
- **Dates and times**: a group that begins a date (#8.6) or a time (#8.5) is not part of it:
  `2500 2026-12-25` gives `2500` and a date, `+2500 2026-12-25` a signed number and a date,
  `06 12 34 15:30` numbers and a time.
- **Right boundary**: the sequence must not be followed, attached, by a letter, a currency, or a
  `.` or a `-` itself followed by a digit: `+15000000F` and `+$15000000` are amounts;
  `06 12 34 56 78-5` ends with a subtraction (the prefix `06 12 34 56` stays an 8-digit phone
  candidate, while `77 123 45 67-5` is rejected entirely because `77 123 45` has only 7 digits). A
  final `.` not followed by a digit stays outside (`Call +1 555 123 4567.`).
- **Last group**: its last group must not begin a longer number (#8.1, #8.2), an amount (#8.3), or a
  percentage (#8.4):
  - `020 7946 0958 50 USD` and `06 12 34 56 78 50 €` give the phone candidate followed by the
    amount (`50 USD`, `50 €`);
  - `77 123 45 67 F` and `77 123 45 67,50` (in `fr`) are rejected and end respectively with the
    amount `67 F` and the number `67,50`;
  - `555-123-4567.50` (in `en`) and `555-123-4567%` are rejected.

These rules guarantee that no phone candidate swallows an amount, a date, or a time: an amount never
disappears silently into a number.

### 8.8 Expression

A sequence of operands joined by operators. Operands: number, amount, percentage, variable, tax
name (#12), function call, parenthesized expression.

| Precedence | Forms |
|---|---|
| 1, highest | simple operands, signed numbers and amounts, parentheses, calls; negation of a parenthesis, a call, or a variable |
| 2 | suffixes `²`, `³`, and `%`, applied left to right in written order |
| 3 | power `^`, right-associative |
| 4 | multiplication (`*`, `×`, `x`, `X`, or implicit) and division `/` |
| 5, lowest | addition and subtraction (`+`, `-`, `−`) |

- Consequences: `2^3²` is `2^9` and `2²^3` is `4^3`. An attached sign is part of the number:
  `-2^2` is `(-2)^2`, that is 4.
- `²` and `³` are attached to what precedes them. `%` is attached or preceded by a single
  whitespace; after a number, it first forms a percentage (#8.4).
- Binary operators are left-associative, except `^`. Whitespace around operators is free; the
  whitespace that follows the expression is not part of it.
- `x` and `X` are operators only between two operands (`3x500`, `3 x 500`, `3 x unit`).
- **Sign or subtraction**: `-` is a subtraction when an operand precedes it, a sign otherwise
  (#8.3). `-` before a parenthesis or a variable, with no operand before it, is a negation.
- **Implicit multiplication**: a number immediately followed by a word that names a visible variable
  (`2m`) is a multiplication; without a visible variable, that word is a unit suffix (text). A
  whitespace between the two prevents implicit multiplication.
- `a + b%` is `a × (1 + b/100)` and `a - b%` is `a × (1 - b/100)`; elsewhere, `b%` is `b/100`
  (#14.4).
- Tax names follow their own rules (#12): `a - tva` is not `a - b%`.
- **Function call**: an identifier present in the lexicon's function list (#16.4), immediately
  followed by `(`, at least one argument, and `)`. Each argument is an expression. `max()` and
  `max(1;)` are text: a trailing separator with no argument prevents the call. Arguments are
  separated by `;`, always accepted, or by `,`: in `en`, every `,` is a separator; in `fr`, a `,`
  between two digits is the decimal separator and any other `,` is a separator. The number of
  arguments is checked at evaluation (#14.6). An identifier followed by `(` that is not a known
  function, or whose arguments do not form expressions, is text. A call wins over a variable of the
  same name.
- **Calculation context**: the right-hand side of a definition (#8.9), the expression of an
  operation, value, or result line of a calculation block (#11.1), and the inside of parentheses and
  calls. Elsewhere, the context is **text**.
- In a calculation context, a word placed where an operand is expected is a variable, even if no
  variable of that name is visible (error `unknown_variable`), unless it names a tax or a function.
  Dates, times, and phones are not recognized there: `1/2` is a division.
- In a text context, a word is an operand only if it names a visible variable or a tax, and an
  expression is kept only if it contains at least one binary operator, an implicit multiplication,
  `^`, `²`, `³`, or a function call. `(500)` alone, `19%` alone, or `+5000` alone are not expressions.
- **Depth**: a parenthesis or a function call placed beyond 32 levels of nesting is not recognized.
  A chain of operations of the same level (`1+2-3+…`, `2^3^2`, `5²%`) has no length limit: it forms
  a single flat item (appendix A, #A.7.2). The depth of an expression's tree is therefore bounded,
  whatever the length of the line.

### 8.9 Variable definition

A whole line (after the prefixes) of the form:

```text
[soit | let] identifier = expression
identifier += expression
identifier -= expression
```

`let` and `soit` come from the lexicon (#16.4) and are both always recognized. `==` is not a
definition. `m += 500` equals `m = m + 500` and `m -= 200` equals `m = m - 200`; the variable must
already be visible, otherwise the line is a definition in error (`unknown_variable`).

- `+=` and `-=` are written with no whitespace between the two characters.
- The expression (calculation context) must occupy the whole rest of the line, trailing whitespace
  aside: `m = 100 kilo` and `d = 5km` are not definitions, but text.
- A word that does not satisfy #8.10 cannot be defined: `well-known = 5` is text.
- In `m = m + 1`, the `m` on the right refers to the previous definition: the new one is visible
  only from the next line.

### 8.10 Identifier

A Unicode letter, then letters, marks, digits, or `_`; 1 to 32 characters after NFC. Comparison
after NFC and case folding: `Price` and `price` are the same identifier. An identifier is therefore
always a whole word (#7).

### 8.11 Mention

`@`, preceded by the start of a line, a whitespace, or `(`, followed by a letter then letters, marks,
digits, or `_`; `-` and `.` are admitted inside, followed by a letter or a digit. `_` reads as a
space: `@Sam_Sea` denotes "Sam Sea". A `@` preceded by a word character belongs to an email or to
text.

### 8.12 Tag

`#`, preceded by the start of a line, a whitespace, or `(`, followed by a letter then letters, marks,
digits, or `_`; `-` is admitted inside, followed by a letter or a digit. `#12` is not a tag.

### 8.13 URL

- The test begins on an ASCII letter or digit.
- A domain **label** begins with an ASCII letter, ASCII digit, or mark, then admits those characters and
  the ASCII hyphen. Neighbouring labels are separated by a dot. A **domain with a suffix** ends with
  a whole suffix from the list (#16.8), preceded by at least one other label; the comparison is done
  in Unicode lowercase, at the label boundaries.
- A URL opens with one of these three forms:
  1. `http://` or `https://`, case-insensitive, followed by at least one character;
  2. `www.`, case-insensitive, followed by at least one character;
  3. the longest sequence of labels with a suffix, immediately followed by `/` (`example.com/x/y`,
     `example.com/abc`).
- The URL stops at the first whitespace or the end of the line. Then, as long as there are any,
  these are removed from its end: a character among `. , ; : ! ? " ' » ’ >`; a `)` if the URL has
  more `)` than `(`; a `]` if it has more `]` than `[`. At least one character must remain after the
  scheme or `www.`, or the `/` after the domain. `(example.com/a).` gives the URL `example.com/a`;
  `(example.com/x(y))` gives the URL `example.com/x(y)`.
- A domain with no path is not a URL: `example.com` alone stays text, `www.example.com` is a URL.
- The target of a URL without a scheme is the same URL preceded by `https://`.

### 8.14 Email

A simplified form of RFC 5322:

- The test begins on an ASCII letter or digit. The local part is the longest run of letters, marks,
  ASCII digits, or the characters `._%+-`; it immediately precedes the `@` and does not end with a
  dot.
- After the `@`, the labels read as for a URL (#8.13). The domain is the longest prefix of labels
  that forms a domain with a suffix: `notes@example.com`.
- The recognition consults no server: it does not say the address exists.

## 9. Variables

1. **Definition**: #8.9. A definition line is never an amount.
2. **Scope**: from the line that follows the definition to the end of the notebook, in text order
   (sections, then blocks, then lines). A definition that is an item of a group (#6.4) holds only
   to the last item of that group; after it, the outer definition, if any, becomes visible again.
   Exception: the variable of a result line (`= total`, #11.4) holds to the end of the notebook,
   even inside a group.
3. **Redefinition**: a new definition of the same identifier holds only for the lines that follow
   it.
4. **Dependencies**: an expression may cite only variables visible at its position. Loops are
   therefore impossible.
5. **Name conflict**: an identifier that is also a unit, a multiplier, a currency, a number word, or
   a tax name is accepted. Within its scope, the defined identifier wins over the other meaning. An
   interactive program SHOULD warn at the definition.
6. **Evaluation**: exact rational arithmetic for `+ − × ÷` and integer powers; decimal with 34
   significant digits for roots and fractional powers (#14). A variable carries a currency if its
   value carries one.
7. **Rounding**: a single one, when the value becomes a counted amount (#14.12).
8. **Errors**: full list at #14.11. The line in error is not counted. An error propagates to
   everything that depends on it. No value is invented.

## 10. Groups and totals

These rules apply to the amounts kept by interpretation (#15).

1. The tags of a title apply to all its items.
2. If items have amounts, the group's total is their sum.
3. If the title itself carries an amount, it is a **check total**:
   - if items have amounts, they are the ones counted, the title is not, and a gap between the two
     SHOULD be reported;
   - if no item has an amount, the title's amount is counted.

## 11. Calculation blocks

### 11.1 Line types

- **Operation line**: after the indentation, an operator among `+`, `-`, `−`, `*`, `×`, `/`, `^`,
  or `x`/`X` (when immediately followed by a whitespace or a digit); then optional whitespace and
  a value that begins an expression (#8.8), and possibly a comment.
  Examples: `+ 19 %`, `- tva`, `- vat`, `* 2`, `/ 4`, `^ 2`, `+ 500 fees`, `-300`, `x3`.
- **Value** (start of an expression): number, amount, visible variable, tax name, function call, or
  parenthesis. `+ fees 500` is not an operation line if `fees` is not a variable.
- **Result line**: after the indentation, `=` (but not `==`), then optional whitespace and an
  optional label.
- **Value line**: a line whose content (after an optional bullet) begins with a value. The rest of
  the line, after the expression, is a comment.
- **Item line**: another line that contains at least one amount kept by interpretation (`- fee 2`,
  `- frais 100`).
- **Comment**: any other line (text, group title).

### 11.2 Detection

A block is a calculation block if and only if it contains:

- at least one result line; or
- at least one operation line whose operator is `*`, `×`, `x`, `X`, `/`, or `^`, or whose expression
  ends with `%` or a tax name (#12).

Otherwise, the block follows the usual rules (#6 and #10).

- Detection takes into account the reading of bullets (#11.3) and the variables visible at each
  line: first the results and definitions are collected, in order, then the bullet rule is applied.
- An expression **ends with `%` or a tax** if it is itself a percentage or a tax, or if its last
  operation is the `%` suffix. For another chain, or a negation, its last operand is looked at. A
  `²`, a `³`, a call, or a parenthesis in last position do not count.
- Outside a calculation block, a definition line keeps its role; the other lines read in a text
  context, with their prefix.

### 11.3 Bullet or subtraction

In a calculation block, a line that begins with `-` or `*`, followed by a whitespace then a value,
is:

- a **bullet** if the first value line of the block begins with the same character followed by a
  whitespace;
- an **operation** otherwise (subtraction or multiplication).

The **first value line** of the block is the first line whose content, after an optional bullet
(`-`, `*`, `•`, `–`, or numbered), begins with a value (#11.1). The rule also serves detection
(#11.2): a line `* 2` read as a bullet does not make the block a calculation one.

| Block | Reading | Result |
|---|---|---|
| `1500` / `- 300` / `=` | subtraction | 1200 |
| `costs:` / `- 1500` / `- 500` / `= total` | bullets | 2000 |
| `* 1500` / `* 2000` | bullets, not a calculation block | — |
| `1500` / `* 2` / `=` | multiplication | 3000 |

`-300` (no whitespace) is always a subtraction. `+` is never a bullet.

### 11.4 Calculation

1. The running total starts at zero.
2. The first value or item line adds its value to the total, which is zero. A first operation line
   applies to zero: `-300` gives −300.
3. An operation line applies its operator to the running total; a value or item line with no
   operator adds to it.
4. An item line contributes the sum of its kept amounts. The block's result takes the weakest zone
   of its item lines (#15.2).
5. A result line shows the running total (a **subtotal**); the calculation then continues from that
   value.
6. If the label of a result line is a single identifier (`= total`), the result is stored in that
   variable, visible from the next line to the end of the notebook (#9, rule 2). Any other label
   (`= sub-total`, `= June total`, `= total juin`) is text.
7. Comments take no part in the calculation.
8. Different currencies in one block raise the error `mixed_currencies`.

The exact detail of the calculation is at #14.9.

### 11.5 Counting

A calculation block counts as **a single amount**: the value of its last result line, or the
running total at the end of the block if it has none. Its lines and its subtotals are never counted
separately.

### 11.6 Example

```text
1500
+ 2500
- 300
= subtotal
* 2
+ vat
= total
```

With a tax `vat` at 19%: subtotal 3700; 3700 × 2 = 7400; 7400 × 1.19 = 8806; total 8806, stored in
the variable `total`. The block counts as a single amount of 8806. The same block with
`= sub-total` and `+ tva` gives the same result for a tax `tva` at 19%.

## 12. Taxes

1. Taxes are settings of the notebook: a name, and a series of rates, each with its effective date
   (#14.1).

2. Common names: `vat`, `tva` (French), `tax`. Other names may be defined (`tva_b`, `sales_tax`). A
   tax name follows the rules of an identifier (#8.10).

3. The rate applied is the one in force at the effective date of the line (#15.1).

4. In an expression, with `t` the rate as a fraction (`19 %` gives `0.19`):

   | Form | Value |
   |---|---|
   | `a + T` | `a × (1 + t)` |
   | `a - T` | `a ÷ (1 + t)` (the tax taken out of the inclusive amount) |
   | `a * T`, `a × T`, `a x T` | `a × t` (the tax amount) |
   | `T` alone | `t` |

5. The **line tax** is the tax amount of the forms `a + T` (`a × t`) and `a - T`
   (`a − a ÷ (1 + t)`), summed over the line (#14.4).

6. A variable defined with a tax's name masks it within its scope (#9, rule 5).

## 13. Fixed-point entry

A notebook setting, off by default. It lets you enter cents without typing the decimal separator.

1. It applies to numbers written with no decimal separator and no multiplier, which are:
   - values added or subtracted in a calculation block (value lines, item lines, `+` and `-`
     operations);
   - or numbers kept as amounts by interpretation.
2. These numbers are divided by `10^d`, where `d` is the number of decimals of the currency
   (#16.9): `1250` gives 12.50 for `d = 2`.
3. It does not apply to the values of a multiplication, a division, or a power, to percentages,
   quantities, times, dates, and years, nor when `d = 0`.

Exact rule: #14.10.

## 14. Evaluation

Evaluation computes, from the tree (appendix A), the values the tree does not contain: variables,
expressions, taxes, calculation-block totals, and the counted amount of each calculation block. It
decides nothing probabilistic: the outside decisions are given to it (#15).

### 14.1 Inputs

1. The **tree** of the whole notebook, with the variable references resolved.

2. The **settings**:

   | Setting | Content |
   |---|---|
   | regional setting | BCP 47 language-country label (`en-US`, `en-GB`, `en-IN`, `fr-FR`, `fr-SN`); see #16.1 |
   | block formats | a block's own regional setting, by the number of its first line |
   | currency | ISO 4217 code of the notebook's currency |
   | taxes | for each tax name: a list of rates, each with its effective date |
   | fixed point | on or off (#13) |
   | import day | the day given to lines placed before the first separator line (#6.2, #15.1) |

   A file's header replaces the settings it gives (#6.6).

   A rate is written in percent (`18` for 18%). The rate of a tax at a date is the one whose
   effective date is the most recent without exceeding that date.

3. The outside **decisions**, line by line (#15): effective date, amounts kept on item lines.

### 14.2 Values

- A value is an **exact fraction**, with an optional currency. The numerator and denominator of the
  reduced fraction must stay below 10^1000 in absolute value; beyond that, the value is the error
  `out_of_range`.
- **Written form**:
  - if the denominator has no prime factors other than 2 and 5: canonical decimal (`.` decimal, no
    thousands separator, no needless zero, `-` for a negative): `8806`, `-12.5`;
  - otherwise: `numerator/denominator`, reduced, the sign on the numerator: `100/3`;
  - the currency follows, after a whitespace: `11.5 USD`, `12.5 EUR`, `100/3 JPY`, `100/3 XOF`.
- **Currencies**:
  - a value carries a currency if one of the operands of the operation carries one;
  - two different currencies in one operation (including a function call, or a block's running
    total and a line's value): error `mixed_currencies`, but mixing different forms of the same
    currency is allowed (`11.5 USD + 100 $`);
  - a number with no currency takes the notebook's currency only when it becomes a counted amount
    (#14.12).

### 14.3 Expressions

Operands:

| Item | Value |
|---|---|
| number | its value |
| amount | its value, with its currency if it is written |
| percentage | its value ÷ 100 (for `a + b%` and `a - b%`, see #14.4) |
| variable | the value of its visible definition; none: error `unknown_variable` |
| tax | the tax's rate at the effective date of the line, as a fraction; no rate: error `no_tax_rate` |
| parentheses | the value of the inner expression |
| call | #14.6 |

Operations:

| Operation | Value |
|---|---|
| `+`, `-` | addition, subtraction; special cases at #14.4 |
| `*` | multiplication, implicit included (`2m`) |
| `/` | division; zero divisor: `division_by_zero` |
| `^` | power (#14.5) |
| `²`, `³` | power 2, power 3 |
| `%` | value ÷ 100 |
| negation | opposite |

A chain of operations of the same level is computed as follows, and the first error met is the
expression's:

- `+ - * /`: left to right, each operand evaluated just before its operation;
- `^`, right-associative (`2^3^2` is `2^9`): all operands left to right, then the powers from the
  end;
- suffixes `² ³ %`: the operand, then each suffix in order.

A chain is computed in one loop: the evaluation stack depends only on the nesting of parentheses and
calls, bounded at 32 levels (#8.8).

### 14.4 Percentages and taxes after `+` and `-`

When the right operand of a `+` or a `-` is **directly** a percentage (a percentage item, or a
chain whose last operation is `%`) or a tax, with `a` the value of the chain before that operation:

| Form | Value | Line tax |
|---|---|---|
| `a + b%` | `a × (1 + b/100)` | — |
| `a - b%` | `a × (1 − b/100)` | — |
| `a + T` | `a × (1 + t)` | `a × t` |
| `a - T` | `a ÷ (1 + t)`; if `1 + t` is zero: `division_by_zero` | `a − a ÷ (1 + t)` |

Elsewhere, a percentage is `b/100` and a tax is `t`: `100 + 2 * 5%` is 100.1; `100 * vat` is
`100 × t`. The **line tax** is the sum of the tax amounts of the `a + T` and `a - T` forms of the
line; it is given only if the line is not in error.

### 14.5 Powers

With `x` the base and `y` the exponent:

- `y` integer: exact computation. `0^0` is 1. `0^y` with `y < 0`: `division_by_zero`. A result out
  of range (#14.2) is detected before the computation.
- `y` non-integer, `x < 0`: error `domain`.
- `y` non-integer, `x = 0`: 0 if `y > 0`, `division_by_zero` otherwise.
- `y` non-integer, `x > 0`, with `y = p/q` reduced:
  - if `q = 2` and `x^p` is within range: square root of `x^p` (#14.7);
  - otherwise: `exp(y × ln x)`, computed with an absolute error on `y × ln x` below 10^-60, then
    rounded to 34 significant digits, to nearest, ties to even.

### 14.6 Functions

Each function has a canonical English name and one or more spellings (#16.4).

| Function | Arguments | Value |
|---|---|---|
| `round` | 1 or 2 | rounded to `n` decimals (0 by default), half away from zero; `n` must be an integer with no currency, from −1000 to 1000 |
| `floor` | 1 | the greatest integer less than or equal |
| `ceil` | 1 | the smallest integer greater than or equal |
| `abs` | 1 | absolute value |
| `min`, `max` | 2 or more | the smallest, the greatest argument |
| `sqrt` | 1 | square root (#14.7); negative argument: `domain` |

- A wrong number of arguments, or an invalid `n` for `round`: error `arguments`.
- The result carries the currency of the arguments (#14.2). For `round`, only the first one's
  counts.

### 14.7 Square root

Correctly rounded to 34 significant digits, to nearest, ties to even. An exact root of 34 digits or
fewer is therefore exact: `sqrt(2.25)` is `1.5`.

### 14.8 Definitions

- `name = expression`: the value of the expression.
- `name += expression` and `name -= expression`: the value of `p + expression` or `p - expression`,
  where `p` is the value of the previous visible definition, with the rules of #14.4. No previous
  definition: `unknown_variable`.
- `= name` (result line): the running total of the block at that line (#14.9).
- Fixed-point entry does not apply to definitions.

A definition in error keeps its error: each line that cites the variable has the same error, with
the same originating line.

### 14.9 Calculation blocks

The running total starts at 0, with no currency. The lines of the block are read in order:

| Line | Effect on the running total |
|---|---|
| value | adds the value of its expression (#14.10) |
| operation `+`, `-` | `total + v` or `total - v`, with the rules of #14.4 (#14.10 for `v`) |
| operation `*`, `/`, `^` | `total × v`, `total ÷ v`, `total ^ v`; a percentage is `b/100`, a tax `t` |
| result | none; stores the total in the variable if the label is an identifier |
| item | adds the sum of its kept amounts (#14.10, #15) |
| definition | none; the variable is evaluated (#14.8) |
| other | none (comment) |

- An **item** line is a line that is neither an operation, nor a result, nor a value, nor a
  definition, and on which interpretation kept at least one amount.
- The running total is noted after each value, operation, result, and item line.
- As soon as a line is in error, the running total takes that error to the end of the block.
- **Block result**: the total of the last result line, otherwise the running total at the end of
  the block.
- **Block zone**: if it has item lines, the weakest of the zones of their kept amounts (#15.2).
- The **counted amount** of the block is its result, rounded (#14.12).

### 14.10 A value written alone: fixed point and ambiguous separator

A **value written alone** is, in a calculation block, the expression of a value line or of a `+` or
`-` operation, or an amount kept on an item line, when it is a number or an amount (and not a
compound expression). Its reference currency is its own if it is written, otherwise the notebook's;
`d` is the number of decimals of that currency (#16.9).

1. **Ambiguous separator**: if the number is marked ambiguous (#8.1), its separator was read as a
   decimal separator, and `d = 0`, the value is multiplied by 1000: in `fr`, in XOF, or in JPY,
   `1,500` is 1500.
2. **Fixed point**: if the setting is on, `d > 0`, and the number is written with no decimal part
   and no multiplier, the value is divided by `10^d`: in EUR, `1250` is 12.5.

Elsewhere, a number keeps the value of the tree: `+ 2x1250` adds 2500, `* 3` multiplies by 3.

### 14.11 Errors

| Code | Cause |
|---|---|
| `division_by_zero` | division by zero, `0` to a negative power, `a - T` with `1 + t = 0` |
| `mixed_currencies` | two different currencies in one operation (#14.2) |
| `unknown_variable` | variable with no visible definition, `+=` or `-=` with no previous definition |
| `arguments` | wrong number of arguments, or an invalid `n` for `round` |
| `domain` | square root of a negative, non-integer power of a negative |
| `out_of_range` | value out of range (#14.2), counted amount out of range (#14.12) |
| `no_tax_rate` | tax with no rate in force at the effective date of the line |
| `currency` | counted amount in a currency with no minor unit (metals, units of account) |

An error carries its code and the number of the line where it was born. It propagates unchanged:
`x = 1/0` on line 3, then `y = x + 1`, gives `y` the `division_by_zero` error of line 3. A line in
error is not counted, and no value is invented in its place.

### 14.12 Counted amount

- The currency of a counted amount is its value's, otherwise the notebook's.
- The value is rounded once to the minor unit of the currency (#16.9), half away from zero.
- The amount is a signed 64-bit integer in the minor unit; beyond that: `out_of_range`.
- The error of a counted amount (`out_of_range`, `currency`) is born at the block's result line:
  its last result line, otherwise its last line.
- A currency absent from the tables of #16.9, even if the lexicon knows it, gives the error
  `currency`.

## 15. Decisions outside the format

Some deterministic rules rest on decisions that the text alone does not fix. This document treats
them as **inputs**. It does not say how to make them.

### 15.1 Effective date

The effective date of a line is, in order: the date written in the line, if an interpretation keeps
one; otherwise the day of its section (#6.2); otherwise the block's creation date, known to the
application; otherwise the header's day (#6.6) or, failing that, the import day. It serves the tax
rates (#12, #14.1).

### 15.2 Kept amounts and zones

On an item line, an interpretation MAY keep amounts: top-level items of the line (number, amount, or
expression), each with a **zone**: `sure` or `probable`, `sure` being the stronger. Groups (#10),
item lines (#11), and fixed point (#13) depend on them.

### 15.3 What interpretation decides

Outside the format: which numbers are amounts and their nature; dates and times in words; date and
phone candidates; persons not marked with `@`; tasks not marked with a box; categories; the use of
a lone variable in a sentence.

### 15.4 Reader without interpretation

An implementation without interpretation MUST behave as if no decision were made:

- no amount is kept: item lines are comments, groups have no total, fixed point touches only the
  values written in calculation blocks;
- the effective date is the day of the section, otherwise the header's day, otherwise the import day;
- candidates (date, phone) stay what the tree says they are: date candidates, numbers.

Calculation blocks, variables, expressions, and taxes stay fully computable.

## 16. Reference lexicon

The lexicon gives the words the grammar recognizes. The **reference lexicon** below is the one of
the conformance cases (appendix C). A conforming parser MUST recognize at least these entries. It
MAY recognize others; it is then level-2 conforming (#18) only if it can be restricted to the
reference lexicon for the checks.

An implementation that adds words changes the trees of some texts: the lexicon is part of the
inputs of determinism (#2.5). Two programs that exchange computed values SHOULD use the same
lexicon.

### 16.1 Regions and separators

The language of the regional setting (`en`, `fr`) fixes the separators of numbers.

| Language | Thousands | Decimal |
|---|---|---|
| `en` | `,` or a whitespace | `.` |
| `fr` | a whitespace or `.` | `,` |

"A whitespace" denotes one of the four whitespaces of #5.

### 16.2 Date order

The day/month order of numeric dates is day-month-year (`DMY`), except in `en-US`, which is
month-day-year (`MDY`).

### 16.3 CFA franc

`F`, `FCFA`, and `CFA` denote XAF if the regional setting's country is one of: `CM` (Cameroon), `CF`
(Central African Republic), `CG` (Congo), `GA` (Gabon), `GQ` (Equatorial Guinea), `TD` (Chad). In
every other region, they denote XOF.

### 16.4 Keywords and functions

- Definition: `let`, `soit`.

- Functions, canonical name then recognized spellings:

  | Canonical | Spellings |
  |---|---|
  | `round` | `round`, `arrondi` |
  | `floor` | `floor`, `plancher` |
  | `ceil` | `ceil`, `plafond` |
  | `abs` | `abs` |
  | `min` | `min` |
  | `max` | `max` |
  | `sqrt` | `sqrt`, `racine` |

### 16.5 Multipliers

| Word | Value |
|---|---|
| `k`, `K`, `thousand`, `mille` | 10³ |
| `M`, `millions`, `million` | 10⁶ |
| `Md`, `bn`, `billion`, `billions`, `milliard`, `milliards` | 10⁹ |

`k`, `K`, and `M` compare case-included; the others after case folding.

### 16.6 Currency symbols

| Symbol | Currency |
|---|---|
| `€` | EUR |
| `$` | USD |
| `£` | GBP |
| `₦` | NGN |
| `₵` | GHS |
| `¥` | JPY |
| `₹` | INR |
| `₣` | CFA franc (#16.3) |

### 16.7 Currency names and codes

- Names, compared after case folding: `f`, `fcfa`, `cfa` (CFA franc, #16.3); `euro`, `euros` (EUR);
  `dollar`, `dollars` (USD); `naira` (NGN); `cedi`, `cedis` (GHS).
- ISO 4217 codes, compared case-included: `EUR`, `USD`, `GBP`, `XOF`, `XAF`, `NGN`, `GHS`, `JPY`,
  `INR`, `TOP`, `ALL`. `TOP` and `ALL` show the case rule: `top 10` and `500 all` are not amounts.

### 16.8 Public suffixes

A subset of the Public Suffix List used by URLs and emails: `com`, `org`, `net`, `io`, `be`, `fr`,
`sn`, `co.uk`. An implementation MAY use the full list; the list's version is then part of its
lexicon.

### 16.9 Minor units (ISO 4217)

The number of minor units (decimals) for each currency, per the ISO 4217 list published on 2026-09-17:

0 decimal:

```text
BIF CLP DJF GNF ISK JPY KMF KRW PYG RWF UGX UYI VND VUV XAF XOF XPF
```

2 decimals:

```text
AED AFN ALL AMD AOA ARS AUD AWG AZN BAM BBD BDT BMD BND BOB BOV BRL BSD BTN BWP
BYN BZD CAD CDF CHE CHF CHW CNY COP COU CRC CUP CVE CZK DKK DOP DZD EGP ERN ETB
EUR FJD FKP GBP GEL GHS GIP GMD GTQ GYD HKD HNL HTG HUF IDR ILS INR IRR JMD KES
KGS KHR KPW KYD KZT LAK LBP LKR LRD LSL MAD MDL MGA MKD MMK MNT MOP MRU MUR MVR
MWK MXN MXV MYR MZN NAD NGN NIO NOK NPR NZD PAB PEN PGK PHP PKR PLN QAR RON RSD
RUB SAR SBD SCR SDG SEK SGD SHP SLE SOS SRD SSP STN SVC SYP SZL THB TJS TMT TOP
TRY TTD TWD TZS UAH USD USN UYU UZS VED VES WST XAD XCD XCG YER ZAR ZMW ZWG
```

3 decimals:

```text
BHD IQD JOD KWD LYD OMR TND
```

4 decimals:

```text
CLF UYW
```

No minor unit (error `currency` at counting): the metals and units of account, `XAG XAU XBA
XBB XBC XBD XDR XPD XPT XSU XTS XUA XXX`, and any code not listed above.

## 17. Reserved symbols and versions

In `dotmemo/1`, the following forms are text, but a future version of the format may give them a
meaning:

- `>` at the start of a line;
- `!` at the start of a line, outside the header (#6.6);
- `{`, `}`, `|`.

A `dotmemo/1` reader MUST keep them as text (#4.2). An author SHOULD escape them (`\>`, `\{`) when
they want them to stay text in every version.

The version is read in the header (#6.6). A file with no header is read as `dotmemo/1`. A new
version of the format does not change the stored text: it changes how it is read. Moving from one
version to another SHOULD show the person the lines whose reading changes.

## 18. Conformance

### 18.1 Levels

| Level | Name | Requirements |
|---|---|---|
| 1 | preserving reader | #4 and #5: import, byte-for-byte preservation, reserved symbols |
| 2 | parser | level 1, and the tree of each syntax case of appendix C, in the format of appendix A |
| 3 | evaluator | level 2, and the results of each evaluation case of appendix C, in the format of appendix B |

An implementation declares its level and the format version: "conforming `dotmemo/1`, level 2".

### 18.2 Verification

- **Syntax**: for each case, parse the text with the reference lexicon (#16) and the case's
  settings, produce the tree in the format of appendix A, complete it with the default values
  (#A.8), and compare it to the expected tree completed likewise. The order of an object's keys does
  not matter.
- **Evaluation**: for each case, parse the text as above, evaluate with the case's settings and
  decisions, and compare to the expected results (appendix B).
- The cases are normative (#3.1). A case that reveals an ambiguity in the text is resolved by fixing
  the document's text.

### 18.3 Required properties

Beyond the cases, a conforming parser MUST have these properties, for any text:

- **lossless** (#4.3);
- **totality**: no text causes a failure or a crash;
- **determinism** (#2.5);
- **bounded time**: the parse time of a line is linear in its length, and that of a text linear in
  its number of lines;
- **bounded depth** (#8.8): nesting is limited to 32 levels; an item counts for at most about 200
  tree levels, and a reader of the JSON tree MUST accept 400 levels of nesting;
- **no swallowed amount**: no phone candidate swallows a group of 4 or more digits with no leading
  zero, a date, or a time (#8.7).

It SHOULD be tested by fuzzing and on large volumes of real short texts.

## 19. Security and privacy

- **Hostile inputs.** A `.dotmemo` file can come from anywhere. The properties of #18.3 bound the
  time and memory of the parse; the limits of #14.2 (10^1000), #14.5 (bounded precision), and #14.12
  (64 bits) bound the evaluation. An implementation MUST NOT use a regular expression with unbounded
  backtracking on the text.
- **Links.** A recognized URL (#8.13) is data. A program SHOULD NOT open, preview, or resolve it
  without the person's action: that would reveal the reading of the file to a third party. The
  display SHOULD show the real domain (beware of look-alike characters in domain names).
- **Personal content.** A notebook holds personal data: names, phone numbers, emails, amounts. The
  format does not protect them: encryption and sharing belong to storage, outside this document. A
  program that recognizes a phone or an email SHOULD NOT send it to an outside service without the
  person's action.
- **Invisible characters.** The text is kept as is (#4.2), direction control characters and
  invisible whitespace included. A display SHOULD make visible those that change the apparent
  meaning of a line.
- **Computed values.** Values are never written into the text (#4.2): a file cannot force a false
  total on its reader.

## 20. Complete example

## 21. Open questions

This draft leaves the following questions open. Feedback is welcome.

1. **Media type.** No media type is registered. A type `text/x-dotmemo` with a `charset=utf-8`
   parameter is the provisional usage.
2. **Lexicon beyond the reference.** Should versioned lexicons wider than the reference lexicon be
   published, so that two implementations can announce the same one?

## Appendix A. Syntax tree in JSON

## Appendix B. Evaluation results in JSON

## Appendix C. Conformance cases

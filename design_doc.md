# Constraints

- take input on STDIN and produce formatted output on STDOUT
    - not 100% sure this is binding but seems like the simplest way to integrate it as an external tool

- total execution time should stay small, ideally under 1 second for a large query

- be installable on common developer machines without major brain damage (ideally packaged as a single executable)


# Non-constraints

- no need to be "scalable"
    - largest likely input is perhaps a few KB of text
    - even a *very* large query won't tokenize to more than a few thousand tokens

- no need to be single-pass
    - per the above, all input comfortably fits in memory
    - materializing lazy sequences into lists is OK
    - making extra copies is OK
    - accumulating output into a buffer and printing once at the end would be OK
        - perhaps after a post-processing pass

- no need to support other formatting styles
    - formatters for other styles exist
    - whole point is to do it MY WAY
    - no intent to support commas-last
    - no intent to support block-indentation style formatting approaches

- no need to support every statement
    - SELECT, INSERT, UPDATE, DELETE, CREATE TABLE are all very valuable
    - probably ignore everything else (pass-through)


# Unknowns

- dialect-specific features
    - how much do we need to handle them?
    - example: the Postgres cast operator `::` may need special handling, but hopefully just won't be present (as a token) in dialects where it's not used
    - odd function forms like `max(foo) keep(dense_rank last order by bar)` or `percentile_disc(0.99) within group(order by response_time)` can hopefully be handled generically
    - dollar-quoting?
    - backtick-quoting?
    - Snowflake's flatten() uses an arrow operator (-> or =>, not sure), not necessary to lay this out smartly but can we at least avoid choking on it?
    - Oracle's `(+)` operator? meh probably not
    - Postgres' array constructor: `array[1,2,3]`
        - and array subscript syntax: `foo[1]` or `bar[2][3]`

- very long function calls: splitting/aligning
    - sometimes seen with window functions?
    - example: `lead(page, 1) over(partition by user_id, request_time::date, some_other_thing order by request_time, and_something_else, a_third_thing)` ideally line-breaks before `order by` and aligns underneath `partition by`, and if the partitioning/ordering expression lists are sufficiently long they should be broken onto lines as well
    - likely requires explicit handling of window syntax
    - not a critical feature; needing to hand-tweak stuff like this after auto-formatting is OK

- long on() clauses and compound predicates? (similar to long function calls)
    - detecting this is hard because it requires looking forward an unknown number of tokens of unknown types
    - probably want to split/align only once the whole expression gets beyond a certain length, which would require accumulating all the tokens before making the layout decision
    - not a critical feature for basic usefulness but would save a lot of time hand-formatting complex queries

- case expressions (again similar)
    - should a multi-line layout always be performed, or should it be length-sensitive?
    - in a multi-line layout, alignment of and/or predicates should match that of `where`
    - the presence/absence of "else null" should not be meddled with, writing it out can add clarity even though it's the default

- line comments
    - some dialects treat `#` as equivalent to, or a substitute for `--`, do we need to support that? maybe a CLI flag?
    - an important part of formatting is putting line breaks where they belong, but line comments are line-break sensitive (obviously)... are there any scenarios where moving a token to a different line changes whether it's commented?
    - similarly, if we move a comment, specifically a real *comment* rather than disabled code, can that create confusion by distancing it from the code it's commenting on?
    - enforce a space before `--`?

- block comments
    - a block comment is inherently pre-formatted, we shouldn't change it
    - also I think the lexer gives it to us as ['/*', 'the whole comment as one token with embedded newlines', '*/']
    - maybe anchor it to the left margin?
    - maybe there's a smarter handling strategy based on whether it starts on its own line, or at the end of a line with other tokens?
    - maybe there's no good solution?

- aliasing
    - some desire to support inserting auto-generated column aliases
    - potential to change semantics, especially when done to CTEs/inline views
    - inserting table aliases is also desirable but harder
        - column references qualified with the full table name would need to be changed to the alias
        - not possible to add the alias qualification to un-qualified column references without knowledge of the schema
    - definitely possible and safe: enforcing the use of `as` in column aliases and removing it from table aliases
    - also possible: right-aligntment of column aliases
    - I prefer aliases to be capitalized (when not double-quoted), but it may be desirable to make this switchable

- should the `outer` keyword be automatically stripped from `left outer join`?   

- how to lay out `cross join`? 

- can/should we automatically add `1=1` to the `where` clause if not present?

- how to handle the set operators: UNION (ALL), INTERSECT, EXCEPT/MINUS ???

- subqueries (both correlated subqueries and inline views)
    - initially I started tracking paren depth to differentiate commas-as-function-arg-separators from commas-as-expression-list-separators
    - being inside parens and encountering SELECT is also significant, and needs to be tracked in order to get indentation correct
    - problem is subqueries are expressions and can be embedded anywhere, e.g. (foo + (select 1) + bar), and the paren depth needs to be remembered and then restored upon exiting the subquery context (suggests a stack)
    - for inline views we can set the left margin just by knowing the nesting depth adding padding before every newline
    - for correlated subqueries that doesn't work because SELECT could be at any arbitrary distance from the left margin and we need to align to it
        - good news is this generalizes to the inline view case so really we only need to solve *this* problem
        - bad news is it means knowing how far we are from the left margin at all times

- how to enforce the use of parens with ON/USING?
    - e.g. `on x.foo = y.foo` should always be `on(x.foo = y.foo)` even though the syntax does not require this
    - inserting parens requires knowing the boundaries of expressions

- tabs to spaces conversion?


# Basic layout

- strip all tokens that are entirely whitespace, since we will be handling spacing and newlines
    - it _may_ be desirable to preserve whitespace that occurs immediately before a block comment

## SELECT

- initial `select` starts at column 1, the `t` in column 6 becomes the alignment ruler for the major clause keywords

- if present, `distinct` (or `all`) immediately follows `select` and pushes the first select expression to the next line

- for each select expression
    - for the first:
        - if DISTINCT or ALL, bump to next line and left-pad with 7 spaces
        - otherwise immediately follow `select` plus a space
    - for all subsequent, add a newline then left-pad with 5 spaces, a comma, and another space (`     , `), aligning the comma under the `t` in `select`
        - note the comma is present as Token.Puncutation<,> and will need to be consumed; this token marks the boundary between expressions
    - print to the end of the line, including `as` and the alias if present

## FROM/JOIN

- left-pad `from` with 2 spaces, aligning the `m` in `from` under the `t` in `select`

- the first table in the `from` clause immediately follows, including alias if present, then a line break

- if tables are listed in `from` SQL-92 style, lay them out as per select expressions above, beginning each line with spaces and a comma aligned under `t` and `m`

- if SQL-99 `join` clauses are used, align `join` or `left` directly under `from`
    - (if some jerk uses `right join` align the `r` under `f` which is terrible but whatever)
    - print the `on` or `using` clause out to the end of the line
    - the next `join|left|right` token marks the boundary
    - `where` marks the end of the from/join scope

## WHERE

- left-pad `where` with 1 space, aligning the `e` in `where` under the `t` in `select`

- the first predicate immediately follows on the same line

- before each `and` or `or` insert a line break and 3 or 4 spaces respectively

- treat parentheses-wrapped compound expressions as a single expression and print on a single line (TODO: specify breaking/alignment for long compound predicates)

## GROUP BY

- left-pad `group by` with 1 space, aligning the `p` in `group` under the `t` in `select`

- lay out group-by expressions as per select expressions, but note that the line of commas aligns to the `y` in `by`!

## HAVING

- `having` prints at column 1 (no padding)

- lay out having expressions _exactly_ as per where expressions

## ORDER BY

- lay out _exactly_ as per group-by expressions

## LIMIT, OFFSET

- left-pad `limit` with one space, print its argument on the same line

- `offset` is the same but with no padding

## CTEs

- `with` prints at column 1 (no padding), followed by the name, followed by `as`, followed by a line break

- then a single left paren in column 1

- the contents of a CTE will be a `select` statement (TODO: support INSERT/UPDATE/DELETE in CTEs, which are a thing in Postgres), which will be laid out exactly as if it were not inside a CTE, except that the left margin will be moved over by N spaces 
    - N has typically been 2 or 4, I'm not sure which I prefer, and it should be easy to specify this as a CLI argument
    - all absolute positioning described here (e.g. "left pad with 2 spaces") is *relative to this margin*

- followed by a single right paren in column 1

- if additional CTEs are present, follow the above closing parent with a line break, then a comma in column 1, the name, `as`, and then repeat the CTE layout starting from the left paren

## Inline Views

- in the from/join scope, a table may be replaced with a `select` statement surrounded by parentheses

- print the opening/left paren on the same line as `from|join|left` and follow with a line break
    - I have sometimes preferred to start the enclosed statement on the same line as the opening paren, so perhaps make that switchable

- the contents of the inline view will be laid out exactly as a bare `select` statement, with the left margin moved over so that the `s` in `select` is one column to the right of the left paren (column 9, for the first level of nesting, or column 14 in the case of `left join`)

- print the closing/right paren in the same column as the opening/left paren, on its own line immediately following the last line of the enclosed statement

- print the `on|using` clause immediately following the closing/right paren, on the same line, exactly as if it were following a from/join identifier

- note that inline views can be nested arbitrarily deep

## Statement terminator

- the trailing semicolon, if present, prints in column 1 on its own line


# Special cases

- `where 1=1` is preferred, though this would normally lay out as `where 1 = 1`
    - probably easiest to fix with post-processing


# Notes on "postgres" lexer behavior

- `x.foo` lexes to:
    - Token.Name<x>, Token.Literal.Number.Float<.>, Token.Name<foo>
    - may need specific handling for the `.` token

- some functions like `greatest` lex to Token.Keyword despite not being keywords

- phrases
    - `group by` and `order by` lex as two tokens, not one
    - `is not null` lexes as three tokens
    - basically assume any N-word keyword phrase lexes as N tokens
    - might be worth modifying the lexer if this turns out to be super annoying

- `'string literal'` lexes to:
    - (Token.Literal.String.Single, "'"), (Token.Literal.String.Single, 'string literal'), (Token.Literal.String.Single, "'")
    - ugh

- `'don''t` lexes to:
    -  (Token.Literal.String.Single, "'"), (Token.Literal.String.Single, 'don'), (Token.Literal.String.Single, "''"), (Token.Literal.String.Single, 't'), (Token.Literal.String.Single, "'")
    - blergh

- `$$dollar literal$$` lexes to:
    - (Token.Literal.String, '$'), (Token.Literal.String.Delimiter, ''), (Token.Literal.String, '$'), (Token.Literal.String, 'dollar literal'), (Token.Literal.String, '$'), (Token.Literal.String.Delimiter, ''), (Token.Literal.String, '$')  
    - sweet jesus  


# Ideas

- is it worth splitting the statement's token stream into chunks based on keyword scope, as the first pass?
    - with, select, from/join, where, group, having, order, window, limit, offset
    - does this actually yield simpler sub-problems or does it result in duplication (e.g. of depth tracking)?

- making constants for the keywords / keyword phrases seems to be working very well -- e.g. SELECT = (Token.Keyword, 'select') -- but the tests,
especially for phrases, are wordy and depend on the phrase length being tested
    - is it worth making a class for this, where an instance corresponds to a word/phrase and knows how to test for itself given a token list?
    - the recursive call still needs to specify how many tokens are consumed so maybe it doesn't help much

- would it be worth re-building the initial token list as namedtuples just to better enable both referring to a token as a unit and referring to its type or value?

- random thought: is it worth doing an initial paren-matching pass?
    - this would give us a quick short-circuit, and hopefully the ability to give a good error message, if the input has unbalanced parens
    - for good input, we would replace the stream of 2-tuple (type, value) tokens with 3-tuples of (type, value, [reference to matching paren])
    - then instead of turning the 3-tuple tokens into strings, we turn them into 5-tuples of (type, value, [matching paren], line, indent)
    - ugh maybe they need to be mutable objects instead of tuples...
    - in any case the methodology is to say, i have a closing paren, bump it to the next line and set its indentation equal to the matching opening paren
    - then once you have a list of these tuples/objects, the actual layout step is to just map them to their indicated X (line) and Y (indent) positions

- quick thought on long compound expressions: it might be simpler to just always do the break-and-align style formatting, then check the length of the result and if shorter than [threshold], unwind it by collapsing whitespace

- I initially switched to a recursive solution becuase I couldn't see how to consume multiple tokens in an iterative fashion, but I figured it out. I don't know why it wasn't obvious to me before... perhaps I've gotten so accustomed to using only for-each style loops or while-true loops that my brain just wasn't prepared for it. Anyway, in the recursive pattern we can consume N tokens by calling the next function with `tokenlist[N:]` as the input argument. The iterative isomorphism is to use an index variable with a while loop, and to advance the index variable by N (instead of always by 1 as a for-each loop does).

- The main use case I was pursuing with this program is the initial reformatting of ugly queries handed to me by others. There is a different use case though, which is handling the tedious reformatting necessitated by certain changes made while working on a query. 
    - For example, if you have `and foo.bar in (select ...)`, and the subquery has line breaks and nice alignment, then you change `foo.bar` to `flerb.baz`, the subquery needs to be re-aligned because the first line is now 2 characters further right.
    - Or if you have `join some_table on(...)`, and the on() clause is compound and nicely aligned, and you replace `some_table` with `some.longer_identifier`, the on() clause needs to be re-aligned.
    - So it would be great if you could pass your in-progress query back through the formatter after making one of these kinds of changes...
    - ...but they key is that when you do that, it's not acceptable for the formatter to re-arrange stuff you may have laid out by hand. Typically this is stuff like long select-clause expressions doing string concatenation or arithmetic, multi-level nested function calls, or particularly elaborate `case` expressions. Because these live in the land of "stuff that a formatting algorithm can't anticipate", the author's layout needs to be respected.
    - Conclusion: in order to support this use case, the formatter needs to be *less* opinionated about how select-clause expressions are laid out, and in particular probably needs to preserve certain whitespace in the input (likely any whitespace not related to indenting expression-separator commas).

### 2020-01-01
- Really starting to think it's time to modify the lexer
    - it would be much simpler if things we need to _treat_ as a single token -- group by, 1.5, 'blergh', >=, x.foo -- actually lexed as a single token rather than a series
    - some of these phrases may need to be assembled in a pre-processing pass over the lexer output, rather than in the lexer itself, we'll see
    - also Pygments' postgres lexer marks some things as Keyword which are not keywords, like "greatest", and I would prefer to be able to consistently differentiate between identifiers and true keywords

- Need to write some code that understands expressions
    - doesn't need to be perfect, ok to allow plausible-looking but invalid constructs
    - doesn't need to understand enough to _evaluate_ expressions, really just know where they begin and end, and the nesting level
    - may want to look up the formal expression grammar
        - https://github.com/ronsavage/SQL
    - probably need a whitelist of binary operators: + - / * = <> != >= <= > < || 
        - maybe also >> << ->> & | ^
    - unary operators???
    - don't need a list of functions, assume any identifier used as such -- `foo(1, 2)` -- is a function call
    
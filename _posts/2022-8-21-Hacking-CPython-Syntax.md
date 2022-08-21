---
layout: post
title: "Hacking CPython: Syntax"
---

This post is the first in a series on adding functionality to the CPython interpreter. In it, we implement the almost-equal operator described in [CPython Internals](https://realpython.com/products/cpython-internals-book/). In this first post, we update the CPython tokenizer, parser, and abstract-syntax tree to support our new feature.

### Motivation

[CPython Internals](https://realpython.com/products/cpython-internals-book/) by Anthony Shaw and the [Real Python](https://realpython.com/) team includes some fun tutorials that walk through modifying the CPython interpreter to support new language constructs. This series of posts does not reproduce all of the information in the book. Instead, I hope to augment the information that Shaw provides with my own thoughts and observations.

At the time of writing, I am a complete newcomer to the CPython source. Accordingly, many of my observations may appear obvious to those more familiar with the codebase.

### Expected Behavior

In this series, we implement an almost-equal operator in CPython. This operator, denoted by `~=`, is capable of performing equality comparisons between integer (`int`) and floating-point (`float`) types. When performing a comparison between an `int` and a `float`, the `float` argument is implicitly converted to an `int` before the comparison is performed. With two `int` operands, the operator behaves identically to equality comparison with `==`.

As described in [the book](https://realpython.com/products/cpython-internals-book/), the almost-equal operator should behave as follows:

```Python
>>> 1 ~= 1
True
>>> 1 ~= 1.0
True
>>> 1 ~= 1.1
True
>>> 1 ~= 1.9
True
```

### Adding a Token

Instead of beginning with updating the Python grammar for our almost-equal operator as Shaw does in the book, I elect to start with updating the tokenizer because I believe it represents the simplest possible modification to the Python syntax. All that is required to update the tokenizer is the addition of a token for `~=` in the tokens file (`Grammar/Tokens`):

```
...
COLONEQUAL              ':='
ALMOSTEQUAL             '~='
```

With the tokens updated, we can regenerate the tokenizer:

```bash
make regen-token
```

After regenerating, we see some modifications to the tokenizer in `Parser/token.c`. First, an `ALMOSTEQUAL` token name is added to the global array:

```c
const char * const _PyParser_TokenNames[] = {
    ...
    "ALMOSTEQUAL",
```

Second, a rule to tokenize this syntax is added to the `PyToken_TwoChars()` function:

```c
int
PyToken_TwoChars(int c1, int c2)
{
    switch (c1) {
        ...
    case '~':
        switch (c2) {
        case '=': return ALMOSTEQUAL;
        }
        break;
```

The tokenizer is now prepared to tokenize our almost-equal operator. Next, we must modify the parser so that it knows what to do when it encounters this token.

### Aside: The Grammar File

The Python grammar file (`python.gram`) reflects the transition to the new Parsing Expression Grammar (PEG) parser for Python. This parser replaced the previous LL(1) parser. [PEP 617](https://peps.python.org/pep-0617/) has more details on the parser transition.

In implementing the almost-equal operator, we modify the `compare_op_bitwise_or_pair` expression. 

```
compare_op_bitwise_or_pair[CmpopExprPair*]:
    | eq_bitwise_or
    | noteq_bitwise_or
    | lte_bitwise_or
    | lt_bitwise_or
    | gte_bitwise_or
    | gt_bitwise_or
    | notin_bitwise_or
    | in_bitwise_or
    | isnot_bitwise_or
    | is_bitwise_or
```

`compare_op_bitwise_or_pair` is the name of the _rule_. This merely provides a way to reference this production in other rules of the grammar (and also identifies it to human readers). 

Immediately following the rule is `[CmpopExprPair*]`; this is the _return type_ of the C or Python function that corresponds to the rule. The inclusion of this return type is optional; if it is omitted, `void*` is inferred for C functions and `Any` is inferred for Python functions.

Finally, following the `:` are the rule's _expressions_. When separated by the `|` symbol, these expressions are checked for a match against the syntax of the program in order - this is is a departure from the manner in which the LL(1) grammar was defined. In the case of the `compare_op_bitwise_or_pair` rule, all of these expressions are references to other rules in the grammar, rather than concrete pieces of syntax. If we follow one of these rules, we can determine more about how the grammar works. For instance, the first rule in the list of expressions for `compare_op_bitwise_or_pair` is `eq_bitwise_or`:

```
eq_bitwise_or[CmpopExprPair*]: '==' a=bitwise_or { _PyPegen_cmpop_expr_pair(p, Eq, a) }
```

In this rule, we see some additional features of the Python grammar. The first expression that follows the rule name is `'=='`, meaning that this rule will match against a literal `==` symbol in the source. Following this, we see an example of a _grammar variable_: `a=bitwise_or`. This is a `bitwise_or` expression whose matched syntax is assigned a name in the expression list such that it can be referred to later in a _grammar action_. In this example, `{ _PyPegen_cmpop_expr_pair(p, Eq, a) }` is a grammar action that allows us to specify how an AST node is generated for the matched syntax directly within the grammar definition.

As a final note, you might be wondering why we match against a `bitwise_or` operator as the second expression in the `eq_bitwise_or` rule - why is this not merely the second operand of the `==` expression? For instance, if we are performing a comparison between two variables:

```Python
a: int = 1
b: int = 2
if a == b:
    pass
```

We would expect this second expression to be a `NAME` that we can match against the literal `b` syntax that we observe in the program. However, the `bitwise_or` expression is necessary because the Python syntax must be sufficiently general to allow for complex constructions. For example:

```Python
if a == b | c & d + 1:
    pass
```

Python must be able to parse the expression on the right of the `==` operator and resolve it to a value (at runtime) for comparison against the left operand. If we follow the chain of productions from the `bitwise_or` rule all the way down to `atom`, we find the following expressions:

```
atom[expr_ty]:
    | NAME
    | 'True' { _Py_Constant(Py_True, NULL, EXTRA) }
    | 'False' { _Py_Constant(Py_False, NULL, EXTRA) }
    | 'None' { _Py_Constant(Py_None, NULL, EXTRA) }
    | '__peg_parser__' { RAISE_SYNTAX_ERROR("You found it!") }
    | &STRING strings
    | NUMBER
    | &'(' (tuple | group | genexp)
    | &'[' (list | listcomp)
    | &'{' (dict | set | dictcomp | setcomp)
    | '...' { _Py_Constant(Py_Ellipsis, NULL, EXTRA) }
```

Here we see some literal values like `NAME` and `NUMBER` to match against concrete syntax in the program.

### Updating the Grammar

Now that we understand the syntax of the Python grammar in `python.gram`, we can modify it to support our almost-equal operator. First, we add a new expression for `ale_bitwise_or` under the `compare_op_bitwise_or_pair` rule:

```
compare_op_bitwise_or_pair[CmpopExprPair*]:
    | eq_bitwise_or
    ...
    | ale_bitwise_or
```

Next, we define the rule for this expression:

```
ale_bitwise_or[CmpopExprPair*]: '~=' a=bitwise_or { _PyPegen_cmpop_expr_pair(p, AlE, a) } 
```

Here we see the same familiar features that were described above:
- `~=` is the syntax for our almost-equal operator
- `a=bitwise_or` is a named expression for the right operand of our operator
- `{ _PyPegen_cmpop_expr_pair(p, AlE, a) }` defines the function that generates an AST node for our operator

With that, we can regenerate the parser:

```bash
make regen-pegen
```

Regenerating the parser updates the parser implementation in `Parser/pegen/parse.c`. In particular, we see the following updates to the `compare_op_bitwise_or_pair_rule()` function:

```c
static CmpopExprPair*
compare_op_bitwise_or_pair_rule(Parser *p) {
    ...
    { // ale_bitwise_or
        if (p->error_indicator) {
            p->level--;
            return NULL;
        }
        D(fprintf(stderr, "%*c> compare_op_bitwise_or_pair[%d-%d]: %s\n", p->level, ' ', _mark, p->mark, "ale_bitwise_or"));
        CmpopExprPair* ale_bitwise_or_var;
        if (
            (ale_bitwise_or_var = ale_bitwise_or_rule(p))  // ale_bitwise_or
        )
        {
            D(fprintf(stderr, "%*c+ compare_op_bitwise_or_pair[%d-%d]: %s succeeded!\n", p->level, ' ', _mark, p->mark, "ale_bitwise_or"));
            _res = ale_bitwise_or_var;
            goto done;
        }
        p->mark = _mark;
        D(fprintf(stderr, "%*c%s compare_op_bitwise_or_pair[%d-%d]: %s failed!\n", p->level, ' ',
                  p->error_indicator ? "ERROR!" : "-", _mark, p->mark, "ale_bitwise_or"));
    }
}
```

Most of this is just error-handling boilerplate; the important line is:

```c
if (
    (ale_bitwise_or_var = ale_bitwise_or_rule(p))  // ale_bitwise_or
)
```

where a new function, `ale_bitwise_or_rule()`, is called to determine if the current state of the parser can match against this rule:

```c
// ale_bitwise_or: '~=' bitwise_or
static CmpopExprPair*
ale_bitwise_or_rule(Parser *p)
{
    if (p->level++ == MAXSTACK) {
        p->error_indicator = 1;
        PyErr_NoMemory();
    }
    if (p->error_indicator) {
        p->level--;
        return NULL;
    }
    CmpopExprPair* _res = NULL;
    int _mark = p->mark;
    { // '~=' bitwise_or
        if (p->error_indicator) {
            p->level--;
            return NULL;
        }
        D(fprintf(stderr, "%*c> ale_bitwise_or[%d-%d]: %s\n", p->level, ' ', _mark, p->mark, "'~=' bitwise_or"));
        Token * _literal;
        expr_ty a;
        if (
            (_literal = _PyPegen_expect_token(p, 54))  // token='~='
            &&
            (a = bitwise_or_rule(p))  // bitwise_or
        )
        {
            D(fprintf(stderr, "%*c+ ale_bitwise_or[%d-%d]: %s succeeded!\n", p->level, ' ', _mark, p->mark, "'~=' bitwise_or"));
            _res = _PyPegen_cmpop_expr_pair ( p , AlE , a );
            if (_res == NULL && PyErr_Occurred()) {
                p->error_indicator = 1;
                p->level--;
                return NULL;
            }
            goto done;
        }
        p->mark = _mark;
        D(fprintf(stderr, "%*c%s ale_bitwise_or[%d-%d]: %s failed!\n", p->level, ' ',
                  p->error_indicator ? "ERROR!" : "-", _mark, p->mark, "'~=' bitwise_or"));
    }
    _res = NULL;
  done:
    p->level--;
    return _res;
}
```

Again, most of this function is boilerplate; the important piece is the conditional where the parser checks if it can match am `ALMOSTEQUAL` token **and** a `bitwise_or` expression. If this succeeds, the almost-equal expression is successfully parsed, and `_PyPegen_cmpop_expr_pair()` is called to construct the return value for the function.

### Updating the AST

If we attempt to recompile CPython now, it will fail with the following message:

```
Parser/pegen/parse.c: In function ‘ale_bitwise_or_rule’:
Parser/pegen/parse.c:9313:51: error: ‘AlE’ undeclared (first use in this function)
 9313 |             _res = _PyPegen_cmpop_expr_pair ( p , AlE , a );
```

The parser is capable of parsing our operator, but it fails when we attempt to generate an AST node.

CPython uses [ASDL](http://asdl.sourceforge.net/) to generate an AST programmatically from a concise definition. Abstract Syntax Definition Language (ASDL) is a language-agnostic tool for defining and generating the code to construct an AST. More information is available on [its website](http://asdl.sourceforge.net/).

If we look in `Parser/Python.asdl` we can find the `cmpop` enumeration that describes all of the valid comparison operator AST-node types:

```
cmpop = Eq | NotEq | Lt | LtE | Gt | GtE | Is | IsNot | In | NotIn
```

We can fix the compilation issue by adding an `AlE` type to this enumeration:

```
cmpop = Eq | NotEq | Lt | LtE | Gt | GtE | Is | IsNot | In | NotIn | AlE
```

```bash
make regen-ast
```

With the AST regenerated, we can successfully compile CPython, but if we attempt to use our almost-equal operator we are met with a nasty error:

```python
>>> 1 ~= 1
Fatal Python error: compiler_addcompare: We've reached an unreachable state. Anything is possible.
The limits were in our heads all along. Follow your dreams.
https://xkcd.com/2200
Python runtime state: initialized

Current thread 0x00007fddeab30280 (most recent call first):
<no Python frame>
[1]    127061 abort (core dumped)  ./python
```

This error is raised because we still need to associate the `ALMOSTEQUAL` token with the `AlE` comparison type. Without this, the AST does not have the knowledge that the `ALMOSTEQUAL` token corresponds to the `AlE` comparison. We remedy this by modifying `ast_for_comp_op()` in `Python/ast.c`:

```c
static cmpop_ty
ast_for_comp_op(struct compiling *c, const node *n)
{
    /* comp_op: '<'|'>'|'=='|'>='|'<='|'!='|'in'|'not' 'in'|'is'
               |'is' 'not'
    */
    REQ(n, comp_op);
    if (NCH(n) == 1) {
        n = CHILD(n, 0);
        switch (TYPE(n)) {
            case LESS:
                return Lt;
            case GREATER:
                return Gt;
            case ALMOSTEQUAL:
                return AlE;
```

Once this is resolved, we can successfully recompile CPython. Furthermore, we now have all of the additions we need to parse our almost-equal operator:

```python
>>> import ast
>>> m = ast.parse('1 ~= 2')
>>> m.body[0].value.ops[0]
<ast.AlE object at 0x7f7188233eb0>
```

This snippet demonstrates that we can successfully produce a well-formed AST from an expression that contains our new operator. However, if we attempt to execute code that contains almost-equals, we encounter the same "Fatal Python error" that we saw previously. This is because nothing beyond the parser and AST, namely the compiler, knows how to handle our new expression. This is the topic of the next post.

### References

- [CPython Internals](https://www.google.com/search?q=cpython+internals&oq=cpython+internals&aqs=chrome.0.69i59j46i512j0i512l5j69i60.1628j0j4&sourceid=chrome&ie=UTF-8)
- [PEP 617 - New PEG Parser for CPython](https://peps.python.org/pep-0617/)
- [Zephyr ASDL Home Page](http://asdl.sourceforge.net/)
- [Using ASDL to Describe ASTs in Compilers](https://eli.thegreenplace.net/2014/06/04/using-asdl-to-describe-asts-in-compilers)
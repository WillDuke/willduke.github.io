---
title: "A Poorman's Rust Enum in Python for ThePrimeagen's Lexer"
date: 2023-08-12T22:22:45-04:00
draft: false
---

A couple of months ago, I came across a video where the prolific streamer and Netflix dev known online as the ThePrimeagen [live-coded](https://www.youtube.com/watch?v=4TxOD2GmWU4) [a simple lexer](https://github.com/ThePrimeagen/ts-rust-zig-deez/blob/master/rust/src/lexer/lexer.rs) for an interpreter inspired by the book [Crafting Interpreters](https://craftinginterpreters.com/) by Robert Nystrom. If you've ever looked into what CPython actually does with your source code, you'll know that it parses the code into an abstract syntax tree, converts the tree into bytecode instructions (i.e. those `*.pyc` files usually in a `__pycache__` directory) and runs them on a virtual machine. But before it can do any of that, it has to "lex" the raw text of the file into meaningful tokens like `def`, `True`, etc. I was struck by the elegance of ThePrimeagen's implementation in Rust, and so I set myself a task: how close can I get to the Rust version using Python?[^1]

## (Sort of) Emulating Rust Enums in Python

Even a superficial glance at the [Rust lexer](https://github.com/ThePrimeagen/ts-rust-zig-deez/blob/master/rust/src/lexer/lexer.rs) will give you an appreciation for Rust enums and pattern matching. In Rust, enums can contain data, so enumerating each of the token types that the lexer recognizes is straightforward:

```rust
pub enum Token {
    Ident(String),
    Int(String),

    Illegal,
    Eof,
    Assign,

    Bang,
    Dash,
    ...
    Return,
    True,
    False,
}
```

In Python, it's similarly eas-- uh, oh.

Python enums just don't work this way. `Enum` subclass members are expected to be singletons, so we'll have to use a different abstraction. After some finagling, I landed on `namedtuples` as a suitable alternative.

```python
from collections import namedtuple

class Token(TokenBase):
    Ident = namedtuple("Ident", ["value"])
    Int = namedtuple("Int", ["value"])
    Illegal = namedtuple("Illegal", [])
    Eof = namedtuple("Eof", [])
    Assign = namedtuple("Assign", [])
    Bang = namedtuple("Bang", [])
    Dash = namedtuple("Dash", [])
    ...
    Return = namedtuple("Return", [])
    True_ = namedtuple("True_", [])
    False_ = namedtuple("False_", [])
```

Not nearly as pretty, but the advantage is that each member of `Token` is now a dynamically generated `NamedTuple` that only has a `value` attribute if the token type requires it. So identifiers like `value` in `let value = 5;` are captured in `Token.Ident("value")`, but `return` tokens (`Token.Return()`) don't carry around extra baggage.

What's `TokenBase`, you ask? Well, if you instantiate a member of the `Token` class without the base class, you'll get something like:

```python
>>> Token.Ident("val")
Ident("val")
>>> Token.Dash()
Dash()
```

Close, but not quite as nice as the way that tokens are displayed in the Rust version: `Token.Ident("val")`. `TokenBase` customizes the `__repr__`s for each of the `namedtuple`s to look more like Rust. (It also modifies `__eq__` and `__neq__` so that empty `namedtuples` with different names do not compare equal!) Here's the ugliness to make that work:

```python
class TokenBase:
    def __init_subclass__(cls) -> None:
        def _token_repr(self):
            name = self.__class__.__name__
            return (
                f"Token.{name}({self.value!r})"
                if hasattr(self, "value")
                else f"Token.{name}"
            )

        def _token_eq(self, other):
            return repr(self) == repr(other)

        def _token_ne(self, other):
            return repr(self) != repr(other)

        for name, method in vars(cls).items():
            if name.startswith("__"):
                continue
            method.__repr__ = _token_repr
            method.__eq__ = _token_eq
            method.__ne__ = _token_ne

```

The trick is that by using `__init_subclass__`, the `__repr__`, `__eq__`, and `__neq__` implementations of the `NamedTuple` class attributes on `Token` will be overwritten *when the class is imported*. This way, `Token.Ident(...)` will be modified without ever instantiating `Token`.

The end result is that instantiating members of `Token` will create `NamedTuple` instances that behave a little bit more like Rust `enum`s.

```python
>>> Token.Ident("val")
Token.Ident('val')
>>> Token.Dash()
Token.Dash
>>> Token.Dash() != Token.Return()
True
```

## Shamelessly Copying (Most of) the Lexer

Though Rust is a substantially more complex language than Python and has been designed with completely different tradeoffs in mind, both languages provide similar procedural syntax. What follows is a more or less direct port of the `Lexer` struct and its methods to peek or advance to the next character in the input text.

```python
@dataclass
class Lexer:
    text: str
    position: int = 0
    read_position: int = 0
    ch: str = field(init=False, default="")

    def __post_init__(self):
        self.read_char()
    
    def __iter__(self):
        while (token := self.next_token()) != Token.Eof():
            yield token
        yield token

    def next_token(self):
        ...

    def peek(self):
        if self.read_position < len(self.text):
            return self.text[self.read_position]

    def read_char(self):
        if self.read_position >= len(self.text):
            self.ch = ""
        else:
            self.ch = self.text[self.read_position]

        self.position = self.read_position
        self.read_position += 1

    def skip_whitespace(self):
        while self.ch.isspace():
            self.read_char()

    def read_ident(self):
        pos = self.position
        while self.peek().isalpha() or self.peek() == "_":
            self.read_char()
        return self.input[pos : self.read_position]

    def read_int(self):
        pos = self.position
        while self.peek().isdigit():
            self.read_char()
        return self.input[pos : self.read_position]
```

The `Lexer` starts at position 0 and steps through the text each time `next_token()` is called. If the next token contains multiple characters or skippable whitespace, `next_token()` will advance the `Lexer` position to the end of the last token. The `read_position` will then correspond to the start of the next token (or whitespace).

The `read_ident` and `read_int` functions repeatedly `peek` at the next character in the input text, instructing the lexer to keep reading the next character until the end of the identifer or integer. The functions record the start and end positions so that they can slice into the input string to return a multi-character token.

Python makes it easy to create custom iterables, so I added an `__iter__` method which will make `for token in lexer: print(token)` work with no extra sweat!

## Matching `match` statements

Things get interesting again when trying to implement `next_token()`. The [Rust implentation](https://github.com/ThePrimeagen/ts-rust-zig-deez/blob/master/rust/src/lexer/lexer.rs#L96-L148) uses a single `match` statement to identify each of the tokens defined in the lexer. In the python version, matching on individual characters ports directly:

```python
def next_token(self):
    self.skip_whitespace()

    match self.ch:
        case "{":
            tok = Token.LSquirly()
        case "}":
            tok = Token.RSquirly()
        case "(":
            tok = Token.Lparen()
        case ")":
            tok = Token.Rparen()
        ...

```

In some cases, such as `Token.Bang`, and `Token.NotEqual`, the lexer peeks at the next character to see which token to emit. This ports over nicely as well:

```python
case "!":
    if self.peek() == "=":
        self.read_char()
        tok = Token.NotEqual()
    else:
        tok = Token.Bang()
```

The case for finding identifiers is particularly elegant in the Rust version: the following excerpt matches on any character that contains an alphabetic character or an underscore to find identifiers, and then uses a nested match statement to identify reserved keywords.

```rust
match self.ch {
    ...,
    b'a'..=b'z' | b'A'..=b'Z' | b'_' => {
        let ident = self.read_ident();
        return Ok(match ident.as_str() {
            "fn" => Token::Function,
            "let" => Token::Let,
            "if" => Token::If,
            "false" => Token::False,
            "true" => Token::True,
            "return" => Token::Return,
            "else" => Token::Else,
            _ => Token::Ident(ident),
        });
    },
    ...
}
```

The `..=` is a convenient syntax for creating inclusive ranges that in this case span either the uppercase or lowercase alphabet. Python does not have a direct equivalent, but according to PEP 636, it does have [match conditionals](https://peps.python.org/pep-0636/#adding-conditions-to-patterns)! Here's what the equivalent looks like in Python:

```python
case t if t.isalpha() or t == "_":
    match self.read_ident():
        case "fn":
            tok = Token.Function()
        ...
        case _ as val:
            tok = Token.Ident(val)
case t if t.isdigit():
    tok = Token.Int(self.read_int())
```

Incidentally, this version supports unicode alphabetic characters as well!

Putting it all together, we get:

```python
def next_token(self):
    self.skip_whitespace()

    match self.ch:
        case "{":
            tok = Token.LSquirly()
        case "}":
            tok = Token.RSquirly()
        case "(":
            tok = Token.Lparen()
        case ")":
            tok = Token.Rparen()
        case ",":
            tok = Token.Comma()
        case ";":
            tok = Token.Semicolon()
        case "+":
            tok = Token.Plus()
        case "-":
            tok = Token.Dash()
        case "!":
            if self.peek() == "=":
                self.read_char()
                tok = Token.NotEqual()
            else:
                tok = Token.Bang()
        case ">":
            tok = Token.GreaterThan()
        case "<":
            tok = Token.LessThan()
        case "*":
            tok = Token.Asterisk()
        case "/":
            tok = Token.ForwardSlash()
        case "=":
            if self.peek() == "=":
                self.read_char()
                tok = Token.Equal()
            else:
                tok = Token.Assign()
        case t if t.isalpha() or t == "_":
            match self.read_ident():
                case "fn":
                    tok = Token.Function()
                case "let":
                    tok = Token.Let()
                case "if":
                    tok = Token.If()
                case "false":
                    tok = Token.False_()
                case "true":
                    tok = Token.True_()
                case "return":
                    tok = Token.Return()
                case "else":
                    tok = Token.Else()
                case _ as val:
                    tok = Token.Ident(val)
        case t if t.isdigit():
            tok = Token.Int(self.read_int())
        case "":
            tok = Token.Eof()
        case _:
            tok = Token.Illegal

    self.read_char()
    return tok
```

Not bad!

## Checking our work

The last task is to reimplement the tests. Here's the smaller of the two tests in the reference, taking advantage of the `__iter__` implementation that we added to the lexer:

```python
def test_next_small():
    text = "=+(){},;"
    lexer = Lexer(text)

    tokens = [
        Token.Assign(),
        Token.Plus(),
        Token.Lparen(),
        Token.Rparen(),
        Token.LSquirly(),
        Token.RSquirly(),
        Token.Comma(),
        Token.Semicolon(),
    ]

    for exp, res in zip(tokens, lexer):
        print(f"expected: {exp}, received {res}")
        assert exp == res

>>> test_next_small()
expected: Token.Assign, received Token.Assign
expected: Token.Plus, received Token.Plus
expected: Token.Lparen, received Token.Lparen
expected: Token.Rparen, received Token.Rparen
expected: Token.LSquirly, received Token.LSquirly
expected: Token.RSquirly, received Token.RSquirly
expected: Token.Comma, received Token.Comma
expected: Token.Semicolon, received Token.Semicolon
```

As usual, the full source for this post is available [here](https://github.com/WillDuke/blog-post-code/blob/master/blog_post_code/lexer/_lexer.py).

[^1]: Many contributors [submitted equivalent lexers](https://github.com/ThePrimeagen/ts-rust-zig-deez/tree/master) in a variety of languages for comparison, including [one in python](https://github.com/ThePrimeagen/ts-rust-zig-deez/tree/master/python) that even includes a parser.
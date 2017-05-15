---
layout: post
title: Programming Language - Part 2 - Tokenizer
---

Welcome to part 2 of our programming language series. In this post we are going to look at implementing a tokenizer.

Before we get started however we should probably briefly cover what a tokenizer is and what they do, which in simple terms
takes a string of input and using some grammar rules, splits up the text into tokens which can be used for further steps in the pipeline - namely parsing.

## Example

```var i = 29;```

Would become:

| **Type** | **Value**  |
| ------ | ------ |
| Keyword | var |
| Whitespace |  |
| Identifier | i |
| Whitespace |  |
| Assignment | = |
| Whitespace |  |
| IntegerLiteral | 29 |
| SemiColon | ;  |
| EOF |  |

Depending on your grammar the results can/will differ.

## Getting Started

Let's start with some sample syntax:

```
import SomeLib;

module SomeModule
{
    public class MyClass
    {

    }
}

```
Now let's build our tokenizer to produce our expected output.

To start with let's define our token types:

```c#
public enum TokenType
{
    EOF,
    Error,
    Whitespace,
    NewLine,
    LineComment, // #, //
    BlockComment, // /* */
    IntegerLiteral,
    StringLiteral,
    RealLiteral,
    Identifier,
    Keyword,
    LeftBracket, // {
    RightBracket, // }
    RightBrace, // ]
    LeftBrace, // [
    LeftParenthesis, // (
    RightParenthesis, // )
    GreaterThanOrEqual, // >=
    GreaterThan, // >
    LessThan, // <
    LessThanOrEqual, // <=
    PlusEqual, // +=
    PlusPlus, // ++
    Plus, // +
    MinusEqual, // -=
    MinusMinus, // --
    Minus, // -
    Assignment, // =
    Not, // !
    NotEqual, // !=
    Mul, // *
    MulEqual, // *=
    Div, // /
    DivEqual, // /=
    BooleanAnd, // &&
    BooleanOr, // ||
    BitwiseAnd, // &
    BitwiseOr, // |
    BitwiseAndEqual, // &=
    BitwiseOrEqual, // |=
    ModEqual, // %=
    Mod, // %
    BitwiseXorEqual, // ^=
    BitwiseXor, // ^
    DoubleQuestion, // ??
    Question, // ?
    Equal, // ==
    BitShiftLeft, // <<
    BitShiftRight, // >>
    Dot,
    Comma,
    Semicolon,
    Colon,
    Arrow, // ->
    FatArrow, // =>
    CharLiteral, // '
}
```

You may or may not want to include some/many of these types, depending on what you are envisioning in your syntax. 
You can always start with a simple subset then gradually include more token types.

Next we will create our "token" class:

```c#
public class Token
{
    public TokenType { get; }
    public SourceFilePart { get; }
    public string Value { get; }

    public Token(TokenType tokenType, string content, SourceFileLocation start, SourceFileLocation end)
    {
        TokenType = tokenType;
        SourceFilePart = new SourceFilePart(start, end, content.Split("\n"));
        Value = content;
    }
}
```

Now for the SourceFilePart and SourceFileLocations. These are to keep track of where the token came from within the file, this helps for printing any errors we may encounter
throughout the various stages of the compiler:

```c#
public class SourceFile
{
    public string Name { get; }
    public string Contents { get; }
    public IEnumerable<string> Lines { get; }

    public SourceFile(string name, string source)
    {
        if (source == null)
            throw new ArgumentNullException(nameof(source));

        Name = name;
        Source = source;
        Lines = _source.Split(new [] { "\n", "\r\n" }, options: StringSplitOptions.RemoveEmptyEntries);
    }
}
```

```c#
public class SourceFilePart
{
    public SourceFileLocation Start { get; }
    public SourceFileLocation End { get; }
    public IEnumerable<string> Lines { get; }

    public SourceFilePart(SourceFileLocation start, SourceFileLocation end, IEnumerable<string> lines)
    {
        Start = start;
        End = end;
        Lines = lines;
    }
}
```

```c#
public class SourceFileLocation
{    
    public int Column { get; }
    public int Index { get; }
    public int Line { get; }

    public SourceFileLocation(int column, int index, int lineNo)
    {
        Column = column;
        Index = index;
        Line = lineNo;
    }
}
```

Now we have some basic builing blocks we can take a look at the tokenizer itself.

```c#
public class Tokenizer
{
    private StringBuilder _sb;
    private int _column;
    private int _index;
    private int _line;
    private ErrorSink _errorSink;
    private ISourceFile _sourceFile;
    private ISourceFileLocation _sourceFileLocation;

    private char _ch = _sourceFile.Contents[_index];
    private char _next = _sourceFile.Contents[_index + 1];

    public ErrorSink ErrorSink => _errorSink;

    public IEnumerable<IToken> Tokenize(string sourceFileContent)
    {
        return Tokenize(new SourceFile("n/a", sourceFileContent));
    }
    public IEnumerable<Token> Tokenize(SourceFile sourceFile)
    {
        if (sourceFile == null)
            throw new ArgumentNullException(nameof(sourceFile));

        _sourceFile = sourceFile;
        _builder.Clear();
        _line = 1;
        _index = 0;
        _column = 1;

        return TokenizeInternal();
    }

    private IEnumerable<Token> TokenizeInternal()
    {
        while(!IsEOF())
            yield return ParseNextToken();

        yield return CreateToken(TokenType.EOF);
    }

    // snip...
}
```

Essentially the above forms the basis of our tokenizer, and although the specifics are not shown yet it should give you an idea of 
how we are approaching this problem, simply put, we take our input string (our source file) and while we are not at the end of the file
keep looping through the content character by character until the end, handling the input and generating tokens as we go.

Next we will implement the "CreateToken" function which will be used throughout the tokenizer:

```c#
private IToken CreateToken(TokenType tokenType)
{
    var content = _builder.ToString();
    var startSourceLocation = _sourceFileLocation;
    var endSourceLocation = new SourceFileLocation(_column, _index, _line);

    _sourceFileLocation = endSourceLocation;
    _builder.Clear();

    return new Token(tokenType, content, startSourceLocation, endSourceLocation);
}
```

Now we have a common method to creating tokens during the tokenization process, there are three other small functions we may find useful:

```c#
// This allows us to Peek ahead X number of characters
private char Peek(int ahead)
{
    if (_index + ahead > _sourceFile.Contents.Length - 1)
        return '\0';

    return _sourceFile.Contents[_index + ahead];
}
```

```c#
// This will 'consume' the character and advance us to the next character
private void Consume()
{
    _builder.Append(_ch);
    Advance();
}
```

```c#
// Advances the index and column for the next token
private void Advance()
{
    _index++;
    _column++;
}
```

With the above utility functions we can now start to look at how we handle parsing the actual tokens:

```c#
private Token ParseNextToken()
{
    if (IsEOF())
    {
        return CreateToken(TokenType.EOF);
    }
    else if (_ch.IsNewLine())
    {
        return NewLine();
    }
    else if (_ch.IsWhiteSpace())
    {
        return WhiteSpace();
    }
    else if (_ch.IsDigit())
    {
        return Int();
    }
    else if ((_ch == '/' && (Peek(1) == '/' || Peek(1) == '*')) || _ch == '#')
    {
        return Comment();
    }
    else if (_ch.IsLetter() || _ch == '_')
    {
        return Identifier();
    }
    else if (_ch == '\'')
    {
        return CharLiteral();
    }
    else if (_ch == '"')
    {
        return StringLiteral();
    } 
    else if (_ch.IsPunctuation())
    {
        return Punctuation();
    }
    else
    {
        return Error();
    }
}
```

### Newlines

```c#
private Token NewLine()
{
    Consume();

    _line++;
    _column = 1;

    return CreateToken(TokenType.NewLine);
}
```

### Handling whitespace

```c#
private Token WhiteSpace()
{
    while (!IsEOF() && _ch.IsWhiteSpace())
        Consume();

    return CreateToken(TokenType.Whitespace);
}
```

### Handling numbers

```c#
private IToken Int()
{
    while (!IsEOF() && _ch.IsDigit())
        Consume();

    // Float / Doubles / Decimal / Exponments
    if (!IsEOF() && (_ch == 'f' || _ch == 'F' || _ch == 'd' || _ch == 'D' || _ch == 'm' || _ch == 'M' || _ch == '.' || _ch == 'e'))
    {
        return Real();
    }

    if (!IsEOF() && !_ch.IsWhiteSpace() && !_ch.IsPunctuation())
    {
        return Error();
    }

    return CreateToken(TokenType.IntegerLiteral);
}
```

```c#
private IToken Real()
{
    if (_ch == 'f' || _ch == 'F' || _ch == 'd' || _ch == 'D' || _ch == 'm' || _ch == 'M')
    {
        Advance();

        if (!IsEOF() && ((!_ch.IsWhiteSpace() && !_ch.IsPunctuation()) || _ch == '.'))
        {
            return Error(message: $"Remove '{_ch}' in real number.");
        }

        return CreateToken(TokenType.RealLiteral);
    }

    int preDotLength = _index - _sourceFileLocation.Index;

    if (!IsEOF() && _ch == '.')
    {
        Consume();
    }

    while (!IsEOF() && _ch.IsDigit())
    {
        Consume();
    }

    if (!IsEOF() && Peek(-1) == '.')
    {
        // .e10 is invalid.
        return Error(message: "Must contain digits after '.'");
    }

    if (!IsEOF() && _ch == 'e')
    {
        Consume();
        if (preDotLength > 1)
        {
            return Error(message: "Coefficient must be less than 10.");
        }

        if (_ch == '+' || _ch == '-')
        {
            Consume();
        }
        while (_ch.IsDigit())
        {
            Consume();
        }
    }

    if (!IsEOF() && (_ch == 'f' || _ch == 'F' || _ch == 'd' || _ch == 'D' || _ch == 'm' || _ch == 'M'))
    {
        Consume();
    }

    if (!IsEOF() && !_ch.IsWhiteSpace() && !_ch.IsPunctuation())
    {
        if (_ch.IsLetter())
        {
            return Error(message: "'{0}' is an invalid real value");
        }

        return Error();
    }

    return CreateToken(TokenType.RealLiteral);
}
```

### Handling comments

```c#
private Token Comment()
{
    Consume();

    if (_ch == '*')
        return BlockComment();

    Consume();

    while (!IsEOF() && !_ch.IsNewLine())
        Consume();

    return CreateToken(TokenType.LineComment);
}
```

```c#
private Token BlockComment()
{
    Func<bool> IsEndOfComment = () => _ch == '*' && _next == '/';
    while (!IsEndOfComment())
    {
        if (IsEOF())
        {
            return CreateToken(TokenType.Error);
        }
        
        Consume();
    }

    Consume();
    Consume();

    return CreateToken(TokenType.BlockComment);
}
```

### Handling identifiers

```c#
private Token Identifier()
{
    while (!IsEOF() && _ch.IsIdentifier())
    {
        Consume();
    }

    if (!IsEOF() && !_ch.IsWhiteSpace() && !_ch.IsPunctuation())
    {
        return Error();
    }

    if (_builder.ToString().IsKeyword(_keywords))
    {
        return CreateToken(TokenType.Keyword);
    }

    return CreateToken(TokenType.Identifier);
}
```

### Handling character literals

```c#
private Token CharLiteral()
{
    Advance();

    var escaping = false;

    while ((_ch.ToString() != "'" || escaping)
    {
        if (IsEOF())
        {
            AddError("Unexpected End Of File", Severity.Fatal);
            return CreateToken(TokenType.Error);
        }

        if (escaping)
        {
            escaping = false;
        }
        else if (_ch == '\\')
        {
            Advance();
            escaping = true;
            continue;
        }

        Consume();
    }

    Advance();

    return CreateToken(TokenType.CharLiteral);
}
```

### Handling string literals

```c#
private Token StringLiteral()
{
    Advance();

    var escaping = false;

    // Escaping characters
    while (_ch != '"' || escaping)
    {
        if (IsEOF())
        {
            AddError("Unexpected End Of File", Severity.Fatal);
            return CreateToken(TokenType.Error);
        }

        if (escaping)
        {
            escaping = false;
        }
        else if (_ch == '\\')
        {
            Advance();
            escaping = true;
            continue;
        }
        
        Consume();                  
    }

    Advance();

    return CreateToken(TokenType.StringLiteral);
}
```

### Handling punctuation

```c#
private Token Punctuation()
{
    switch (_ch)
    {
        case ';':
            Consume();
            return CreateToken(TokenType.Semicolon);

        case ':':
            Consume();
            return CreateToken(TokenType.Colon);

        case '{':
            Consume();
            return CreateToken(TokenType.LeftBracket);

        case '}':
            Consume();
            return CreateToken(TokenType.RightBracket);

        case '[':
            Consume();
            return CreateToken(TokenType.LeftBrace);

        case ']':
            Consume();
            return CreateToken(TokenType.RightBrace);

        case '(':
            Consume();
            return CreateToken(TokenType.LeftParenthesis);

        case ')':
            Consume();
            return CreateToken(TokenType.RightParenthesis);

        case '>':
            Consume();
            if (!IsEOF() && _ch == '=')
            {
                Consume();
                return CreateToken(TokenType.GreaterThanOrEqual);
            }
            else if (!IsEOF() && _ch == '>')
            {
                Consume();
                return CreateToken(TokenType.BitShiftRight);
            }
            return CreateToken(TokenType.GreaterThan);

        case '<':
            Consume();
            if (!IsEOF() && _ch == '=')
            {
                Consume();
                return CreateToken(TokenType.LessThanOrEqual);
            }
            else if (!IsEOF() && _ch == '<')
            {
                Consume();
                return CreateToken(TokenType.BitShiftLeft);
            }
            return CreateToken(TokenType.LessThan);

        case '+':
            Consume();
            if (!IsEOF() && _ch == '=')
            {
                Consume();
                return CreateToken(TokenType.PlusEqual);
            }
            else if (!IsEOF() && _ch == '+')
            {
                Consume();
                return CreateToken(TokenType.PlusPlus);
            }
            return CreateToken(TokenType.Plus);

        case '-':
            Consume();
            if (!IsEOF() && _ch == '=')
            {
                Consume();
                return CreateToken(TokenType.MinusEqual);
            }
            else if (!IsEOF() && _ch == '>')
            {
                Consume();
                return CreateToken(TokenType.Arrow);
            }
            else if (!IsEOF() && _ch == '-')
            {
                Consume();
                return CreateToken(TokenType.MinusMinus);
            }
            return CreateToken(TokenType.Minus);

        case '=':
            Consume();
            if (!IsEOF() && _ch == '=')
            {
                Consume();
                return CreateToken(TokenType.Equal);
            }
            else if (!IsEOF() && _ch == '>')
            {
                Consume();
                return CreateToken(TokenType.FatArrow);
            }
            return CreateToken(TokenType.Assignment);

        case '!':
            Consume();
            if (!IsEOF() && _ch == '=')
            {
                Consume();
                return CreateToken(TokenType.NotEqual);
            }
            return CreateToken(TokenType.Not);

        case '*':
            Consume();
            if (!IsEOF() && _ch == '=')
            {
                Consume();
                return CreateToken(TokenType.MulEqual);
            }
            return CreateToken(TokenType.Mul);

        case '/':
            Consume();
            if (!IsEOF() && _ch == '=')
            {
                Consume();
                return CreateToken(TokenType.DivEqual);
            }
            return CreateToken(TokenType.Div);

        case '.':
            Consume();
            return CreateToken(TokenType.Dot);

        case ',':
            Consume();
            return CreateToken(TokenType.Comma);

        case '&':
            Consume();
            if (!IsEOF() && _ch == '&')
            {
                Consume();
                return CreateToken(TokenType.BooleanAnd);
            }
            else if (!IsEOF() && _ch == '=')
            {
                Consume();
                return CreateToken(TokenType.BitwiseAndEqual);
            }
            return CreateToken(TokenType.BitwiseAnd);

        case '|':
            Consume();
            if (!IsEOF() && _ch == '|')
            {
                Consume();
                return CreateToken(TokenType.BooleanOr);
            }
            else if (!IsEOF() && _ch == '=')
            {
                Consume();
                return CreateToken(TokenType.BitwiseOrEqual);
            }
            return CreateToken(TokenType.BitwiseOr);

        case '%':
            Consume();
            if (!IsEOF() && _ch == '=')
            {
                Consume();
                return CreateToken(TokenType.ModEqual);
            }
            return CreateToken(TokenType.Mod);

        case '^':
            Consume();
            if (!IsEOF() && _ch == '=')
            {
                Consume();
                return CreateToken(TokenType.BitwiseXorEqual);
            }
            return CreateToken(TokenType.BitwiseXor);

        case '?':
            Consume();
            if (!IsEOF() && _ch == '?')
            {
                Consume();
                return CreateToken(TokenType.DoubleQuestion);
            }

            return CreateToken(TokenType.Question);

        default:
            return Error();
    }
```

### Error Handling

```c#
private Token Error(Severity severity = Severity.Error, string message = "Unexpected token '{0}'")
{
    while (!IsEOF() && !_ch.IsWhiteSpace() && !_ch.IsPunctuation())
        Consume();

    AddError(string.Format(message, severity), severity);

    return CreateToken(TokenType.Error);
}
```

```c#
private void AddError(string message, Severity severity)
{
    var sourcePart = new SourceFilePart(_sourceFileLocation, new SourceFileLocation(_column, _index, _line), _builder.ToString().Split('\n'));
    _errorSink.AddError(message, sourcePart, severity);
}
```

### Summary

This should now give us the fundimentals of our tokenizer, allowing us to generate tokens which can be used going forward to parsing, 
and generating a simple AST.

I would like to add that during the process of creating your own tokenizer it's generally a good idea to write some unit tests along the way, this will help 
validate that you are generating the tokens you expect from your input and help when making changes or adding different rules, for example when I initially implemented 
escaped string literals I had enough tests to prevent breaking any other part of the tokenizer, and ensured that existing behaviour still matched the desired result.

Stay tuned for part 3 when we start looking at parsing an AST from our tokens!
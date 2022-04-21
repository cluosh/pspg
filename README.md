pspg 
===

**pspg** is an LL(1) parser generator in PostScript, written as final project
of the [stack-based languages course at TU Wien](https://vowi.fsinf.at/wiki/TU_Wien:Stackbasierte_Sprachen_VU_(Ertl))
in WS2018.

What the fuck?
---

Yea. Exactly. I don't know, I found this code rotting on my backup disk. Seems
pretty readable for PostScript standard, but at this point this seems like a
distant fever dream and I have no idea how any of this works anymore. I'm not
sure if I can morally claim ownership of this code, as I'm pretty sure I was 
an entirely different person when I wrote this code.

Introduction
---

The initial goal of this project was to parse [GLSL fragment shaders](https://www.khronos.org/opengl/wiki/Fragment_Shader)
in PostScript and render the output. Since the project wasn't supposed to take
long, I quickly realized that this was out of scope and focused on parsing
GLSL itself, by creating an LL(1) parser generator. The parser generator is
pretty much based on the textbook algorithms making use of
first/follow/transition tables, so nothing special in that, except that it's
written in PostScript. Eventually time ran out, so I added two languages as
examples: [lang_calc](lang_calc), which is a simple calculator language and [lang_glsl](lang_glsl),
which is a sub-sub-sub-sub-sub-subset of actual GLSL (basically the *"oh shit I
have to present this tomorrow, I need a minimal example that actually works"*-subset).

Running the Code
---

The files [calc.ps](calc.ps) and [glsl.ps](glsl.ps) give an overview on how
the parser generator works and are the "executable entrypoints" of this whole
thing, for example in `calc`:

```postscript
(lib/lib.ps) run

% Generate parser
(lang_calc/tokens.ps) run
(lang_calc/grammar.ps) run
PARSER_generate

% Lexical analysis on input file
(lang_calc/examples/example.calc)
LEXER_file LEXER_tokenize

% Parse tokens
exch PARSER_parse
```

This code will first create a parser generator, based on the grammar
definitions in the [lang_calc](lang_calc) folder and then parse the example
file. It can be executed with [GhostScript](https://www.ghostscript.com/) in
the following manner (tested with version 9.56.1):

```bash
[cluosh@nyarlathotep sb]$ gs -dNOSAFER -dNODISPLAY -q calc.ps
GS<1>
```

After the execution of `exch PARSER_parse`, the AST of the example file is
placed on the stack and can be displayed by executing `TREE_print`:

```text
GS<1>TREE_print
  expr
    term
      factor
        LEFT_PAREN
        expr
          term
            factor
              NUMBER
              1
          expr_1
            ADD
            term
              factor
                NUMBER
                2
        RIGHT_PAREN
      term_1
        DIV
        factor
          NUMBER
          3
        term_1
          MUL
          factor
            LEFT_PAREN
            expr
              term
                factor
                  NUMBER
                  4
              expr_1
                ADD
                term
                  factor
                    NUMBER
                    5
            RIGHT_PAREN
GS>
```

Token & Grammar Specification
---

An example for the specification of lexer tokens can be found in
[tokens.ps](lang_calc/tokens.ps) of the `calc` language:

```postscript
/tokens
<<
    /LEFT_PAREN    {}
    /RIGHT_PAREN   {}
    /NUMBER        {}
    /ADD           {}
    /SUB           {}
    /MUL           {}
    /DIV           {}
>> def

/separators
<<
    (+) ord      { (+) LEXER_fix }
    (-) ord      { (-) LEXER_fix }
    (*) ord      { (*) LEXER_fix }
    (\/) ord     { (_DIV_) LEXER_fix }
    (\() ord     { (_LP_) LEXER_fix }
    (\)) ord     { (_RP_) LEXER_fix }
>> def

/keywords
<<
    /_LP_        { [ /LEFT_PAREN ] }
    /_RP_        { [ /RIGHT_PAREN ] }
    /+           { [ /ADD ] }
    /-           { [ /SUB ] }
    /*           { [ /MUL ] }
    /_DIV_       { [ /DIV ] }
>> def

/litAndIdents
<<
    /realtype    { cvr /NUMBER }
    /integertype { cvi /NUMBER }
>> def
```

This could be done better in PostScript (could just use the tokens as actual
PostScript tokens), but it worked I guess. The `separators`, `keywords` and
`litAndIdents` are used by the lexer to produce the `tokens` needed by the
parser. Using the `tokens`, you can then specify the actual grammar, as in
[grammar.ps](lang_calc/grammar.ps):

```postscript
% Define starting symbol
/grammarStart /expr def

% Define grammar
/grammar
<<
    /expr [
        [ /term /expr_1 ]
    ]
    
    /expr_1 [
        [ /ADD /term /expr_1 ]
        [ /SUB /term /expr_1 ]
        [ ]
    ]
    
    /term [
        [ /factor /term_1 ]
    ]
    
    /term_1 [
        [ /MUL /factor /term_1 ]
        [ /DIV /factor /term_1 ]
        [ ]
    ]
    
    /factor [
        [ /LEFT_PAREN /expr /RIGHT_PAREN ]
        [ /NUMBER ]
    ]
>> def
```

Code Structure
---

* `lang_calc` - Contains token/grammar definitions for the `calc` language
* `lang_glsl` - Contains token/grammer definitions for the GLSL subset
* `lib` - Contains the parser generator and parsing code
  + `generator` - Contains generation of first/follow/transition tables
  + `parser` - Uses the generated parser to parse a grammar, AST stuff

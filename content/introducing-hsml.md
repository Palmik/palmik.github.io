+++
title = "Introducing HSML"
date = 2012-08-23

[taxonomies]
categories = []
tags = ["Haskell"]
+++

`HSML` (shorthand for _Haskell's Simple Markup Language_) is simple markup language with syntax similar to `XML` and `HTML`. It allows you to embed Haskell expressions and declarations in your templates.

<!-- more -->

There are two incarnations of `HSML`

*   Classic `HSML` (or just `HSML`): this is the full blown templating system. These templates can contain expressions, declarations and template arguments. They are translated into record type and its instance of [`IsTemplate`](http://hackage.haskell.org/packages/archive/template-hsml/0.2.0.2/doc/html/Template-HSML.html#t:IsTemplate) type class.
    
*   Simplified `HSML`: this is a cut-down version of the full blown system, the only feature that is lacking are template arguments. Simplified `HSML` templates are translated into expression of the type [`Text.Blaze.Markup`](http://hackage.haskell.org/packages/archive/blaze-markup/latest/doc/html/Text-Blaze.html#t:Markup).
    

Valid simplified `HSML` is valid `HSML`. Valid `HSML` with its arguments removed is also valid simplified `HSML`.

You can find `HSML` on [hackage](http://hackage.haskell.org/package/template-hsml) and follow its development on [github](https://github.com/Palmik/HSML).

## Syntax

Valid `HSML` document starts with declarations of the template's arguments, this is the only syntactical difference from simplified `HSML`.

After that follows a list of _chunks_, where _chunk_ is either _text_, _raw text_, _element node_, _element leaf_ or _haskell_, where _haskell_ can be either Haskell expression or Haskell declaration.

What follows is a brief description with examples for every `HSML` construct.

### Text

The most basic _chunk_ is text. Text can contain any characters, except for `<` and `{` which have to be escaped using `\` (in fact, you can use `\` to escape any character). Text is also automatically `HTML` escaped.

#### Syntax

```
? all characters except '<' and '{' which have to be escaped ? 
```

#### Example

```
This is text that ends with less-than sign and opening curly bracket \<\{
```

### Raw Text

Raw text is similar to text, but is rendered as is, that means without `HTML` escaping. Raw text can contain any characters, but can not contain `|}` as a substring (you can circumvent this by escaping).

#### Syntax

```
"{r|" ? all characters, the sequence can not contain "|}" substring ? "|}" 
```

#### Example

```
{r|This text is raw|}, but this is not. Do you want "|}" inside raw text?
No problem, do it like this: {r|foo |\} bar|}
```

### Element node

Element node is an element that can contain other _chunks_. In `HTML` it could be for example `<div>...</div>`. Element nodes can have attributes. Attributes, attribute names and attribute values can also be expressions.

These additional requirements have to be met:

*   The attribute expressions have to be of type [`Text.Blaze.Attribute`](http://hackage.haskell.org/packages/archive/blaze-markup/0.5.1.0/doc/html/Text-Blaze-Internal.html#t:Attribute).
    
*   The attribute name expressions have to be of type `String`.
    
*   The attribute value expressions have to be of a type that is an instance of [`Text.Blaze.ToValue`](http://hackage.haskell.org/packages/archive/blaze-markup/latest/doc/html/Text-Blaze.html#t:ToValue) or of type [`Text.Blaze.AttributeValue`](http://hackage.haskell.org/packages/archive/blaze-markup/latest/doc/html/Text-Blaze-Internal.html#t:AttributeValue), depending on the [template options](http://hackage.haskell.org/packages/archive/template-hsml/0.2.0.2/doc/html/Template-HSML.html#t:Options).
    

#### Syntax

```
"<" element_name { attribute } ">" { chunk } "</" element_name ">" 
```

#### Example

```html
<ul class={h|userID|}>
  <li>Name: {h|userName|}</li>
  <li>Age: {h|userAge|}</li>
</ul>

<div {h|name|}="static_value" static_name={h|value|}>
  This div has attribute with dynamic name and an attribute with dynamic value.
</div>

<div {h|name|}={h|value|} {h|attribute|}>
  This div has attribute with dynamic name and value.
</div>

<div {h|attribute|}>
  This div has fully dynamic attribute. This allows for optional attributes.
</div>
```

### Element leaf

Element leaf is an element that can not contain any other _chunks_. In `HTML` it could be for example `<br/>`. Element leafs can also have attributes.

#### Syntax

```
"<" element_name { attribute } "/>"
```

#### Example

```html
Some things are just broken, <br/>
just like this line.
```

### Haskell

The Haskell _chunks_ in your templates can be either expressions or declarations.

Declarations are scoped, this means that if you have a declaration inside an _element node_, it can be used only inside that node (that also includes any nested _chunks_). Top level declarations (those that are not inside any _element node_) are visible in the whole template.

Expressions have to be of a type that is an instance of [`Text.Blaze.ToMarkup`](http://hackage.haskell.org/packages/archive/blaze-markup/latest/doc/html/Text-Blaze.html#t:ToMarkup) or of type [`Text.Blaze.Markup`](http://hackage.haskell.org/packages/archive/blaze-markup/latest/doc/html/Text-Blaze.html#t:Markup), depending on the [template options](http://hackage.haskell.org/packages/archive/template-hsml/0.2.0.2/doc/html/Template-HSML.html#t:Options).

#### Syntax

```
"{h|" expression | declaration "|}"
```

#### Example

```html
{h| omnipresent = "omnipresent" |}

<div>
  {h| local = "first local" |}
  <p>{h|omnipresent|}</p>
  <p>{h|local|}</p>
</div>

<div class={h|omnipresent|}>
  {h| local = "second local" |}
  <p>{h|omnipresent|}</p>
  <p>{h|local|}</p>
</div>
```

### Argument

Arguments get translated into fields of the record type, you can optionally specify the type of the field. If you decide to not specify the type, the type will become a parameter of the record type.

#### Syntax

```
"{a|" <argument name> [ :: <argument type> ] "|}"
```

#### Example

```
{a| name :: String |}
{a| age :: Int |}
{a| mystery |}

And the mystery was: {h|mystery|}
```

## Usage

If you have come this far, you might be interested in trying out `HSML`. The `Template.HSML` module exports all that you need and I recommend you to read the [documentation](http://hackage.haskell.org/packages/archive/template-hsml/0.2.0.2/doc/html/Template-HSML.html) or ask me to improve it if you happen to find it lacking.

Often the most efficient way to learn is by example, so here is one:

#### Main.hs

```haskell
{-# LANGUAGE TemplateHaskell #-}
{-# LANGUAGE QuasiQuotes     #-}
{-# LANGUAGE RecordWildCards #-}

------------------------------------------------------------------------------
import           Control.Monad
------------------------------------------------------------------------------
import           Data.Monoid ((<>))
------------------------------------------------------------------------------
import qualified Text.Blaze.Html5 as B
------------------------------------------------------------------------------
import           Template.HSML
------------------------------------------------------------------------------


data User = User
    { userID :: Int
    , userName :: String
    , userAge :: Int
    } 

$(hsmlFileWith (defaultOptions "Default") "default_layout.hsml")

homeTemplate :: [User] -> B.Markup
homeTemplate users = renderTemplate Default
    { defaultTitle = "Home page"
    , defaultSectionMiddle = middle
    , defaultSectionFooter = [m| <p>Generated by HSML</p> |]
    }
    where
      middle = [m|
        <ul class="users">
          {h| forM_ users wrap |}
        </ul> |]
      wrap u = [m|<li> {h| userTemplate u |} </li>|]
        
userTemplate :: User -> B.Markup
userTemplate User{..} = [m|
  <ul class={h| "user-" <> show userID |}>
    <li>Name: {h|userName|}</li>
    <li>Age: {h|userAge|}</li>
  </ul> |]
```

#### default\_layout.hsml

```html
{a| title :: String |}
{a| sectionMiddle :: B.Markup |}
{a| sectionFooter :: B.Markup |}

{h| B.docType |}

<html lang="en">
  <head>
    <meta charset="utf-8"/>
    <title>{h|title|}</title>
  </head>

  <body>
    <div class="section middle">
      {h|sectionMiddle|}
    </div>

    <footer>
      {h|sectionFooter|}
    </footer>
  </body>
</html>
```

And the prettified result of `renderMarkup $ homeTemplate [User 1 "Jon Doe" 16, User 2 "Jane Roe" 17]` is this:

```html
<!DOCTYPE HTML>
<html lang="en">
  <head>
    <meta charset="utf-8">
    <title>Home page</title>
  </head>

  <body>
    <div class="section middle">
      <ul class="users">
        <li>
          <ul class="user-1">
            <li>Name: Jon Doe</li>
            <li>Age: 16</li>
          </ul>
        </li>
        <li>
          <ul class="user-2">
            <li>Name: Jane Roe</li>
            <li>Age: 17</li>
          </ul>
        </li>
      </ul>
    </div>

    <footer>
      <p>Generated by HSML</p>
    </footer>
  </body>
</html>
```

## Thoughts

Currently, nesting `HSML` within expressions or declarations within `HSML` (and of course deeper nesting) is not really user-friendly. That's because neither the embedded Haskell expressions nor declarations can contain Quasi Quotes (as `haskell-src-meta` does not support that at the moment -- there is a pending [ticket](https://github.com/benmachine/haskell-src-meta/issues/9) for that). But I think that it should be possible to work around that by transforming [`QuasiQuote`](http://hackage.haskell.org/packages/archive/haskell-src-exts/latest/doc/html/Language-Haskell-Exts-Syntax.html#t:Exp) into [`SpliceExp`](http://hackage.haskell.org/packages/archive/haskell-src-exts/latest/doc/html/Language-Haskell-Exts-Syntax.html#t:Exp) -- I will definitely try it out.

## Epilogue

I hope you liked the article and `HSML`. There are still few things that remain to be done, like better test coverage, measure the impact on run-time performance, consider using `attoparsec` instead of `parsec`, etc.

If you have any comments or ideas for improvement, I will gladly hear them out.

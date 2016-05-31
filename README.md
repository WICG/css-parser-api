# Overview
The goal of this specification is to allow authors access to the engine's parser.
There are two over-arching use cases:
1. Pass the parser a string receive and receive an object
to work with instead of building custom parsing in JS.
2. Extend what the parser understands for fully polyfilling
    
Desires for these APIs:
* Build on top of the [TypedOM](https://drafts.css-houdini.org/css-typed-om/) and make modifications to that
  specification where necessary.
* Be able to request varying levels of error handling by the parser

#Usecase 1 - Sketch of ideas  
Basically what this will allow you to do is something similiar to the current
results of inserting new stylesheet via JS to force it to be within the CSSOM
but get the benefits of it not needing to be an actual stylesheet or iterate
over it to find your result.

### Strawman IDL
```
    partial interface Window {
        void cssParse
    }
    
    partial interface cssParse {
        bool onlyCheckForSyntaxError // This will only check against grammar, not engine support
        StylePropertyMapReadOnly rule(string);
        Sequence<StylePropertyMapReadOnly> ruleSet(string);
    }
```

### Example

```
   var background = window.cssParse.rule("background: green");
   console.log(background.styleMap.get("background").value) // "green"
   
   var styles = window.cssParse.ruleSet(".foo { background: green; margin: 5px; }");
   console.log(styles.length) // 5
   console.log(styles[0].styleMap.get("margin-top").value) // 5
   console.log(styles[0].styleMap.get("margin-top").type) // "px"
```

Use Cases
=========

1. Extending CSS in a more intrusive way than our hooks allow - most of the stylesheet is valid CSS, but you want to intercede on the unrecognized things and fix it up into valid CSS.  (Probably don't want to do this - it collides with the custom property/function/at-rule case, but without the friendliness to future language extensions.)
2. Custom properties that are more complex than we allow you to specify - something that takes "`<length> || <color>`", for example. Right now your only choice is to give it the grammar `"*"` which just gives you a string that you have to parse yourself.  Similar with custom functions (`--conic-gradient(...)`) and custom at-rules (`@--svg`) - these will *rarely* fit in our limited set of allowed grammars.
3. CSS-like languages that want to rely on CSS's generic syntax, like CAS or that one mapping language I forget the name of. These have nothing to do with CSS and will likely result in something other than a stylesheet: CAS turns into a series of querySelector() and setAttribute() calls; the mapping thing turns into some data structures specific to that application.
4. Extending HTML/SVG attributes that use a CSS-like syntax, like `<img sizes>`. If you wanted to add support for the `h` descriptor to `sizes`, you currently have to write your own full-feature CSS parser. (`sizes` is pretty complex; you shouldn't skimp.) Better to let the engine parse it as "generic CSS", then you can recognize the parts you need and do image selection yourself.

Work Items
==========

1. Draw up a proposal for "generic CSS" interfaces; a half-dozen or so tokens that we can generically expose (numbers, dimensions, etc), plus some for the various "containers" (functions, blocks, at-rules).  These will be exposed by the "generic parsing" APIs, as basically a low-powered version of the Typed OM.
2. Expose all of the Syntax parsing hooks on `window.CSS`. Maybe add a few more as needed, like "parse a media query" or something.
3. Write up pseudo-code examples for all of the use-cases, illustrating how the proposed API would be used. The use-cases above all have concrete examples that can be used.

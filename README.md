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
   console.log(background.styleMap.get("background").ruleSet) // "green"
   
   var styles = window.cssParse.ruleSet(".foo { background: green; margin: 5px; }");
   console.log(styles.length) // 5
   console.log(styles[0].styleMap.get("margin-top").value) // 5
   console.log(styles[0].styleMap.get("margin-top").type) // "px"
```


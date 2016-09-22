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

## Example: Parsing out a ruleset

```
   var background = window.cssParse.rule("background: green");
   console.log(background.styleMap.get("background").value) // "green"

   var styles = window.cssParse.ruleSet(".foo { background: green; margin: 5px; }");
   console.log(styles.length) // 5
   console.log(styles[0].styleMap.get("margin-top").value) // 5
   console.log(styles[0].styleMap.get("margin-top").type) // "px"
```

## Example: Parsing out a stylesheet

```
	const style = fetch("style.css")
		.then(response=>CSS.parseStylesheet(response.body));
	style.then(console.log);

    /* example of the object once we have it more refined */
    );
```

Use Cases
=========

1. Extending CSS in a more intrusive way than our hooks allow - most of the stylesheet is valid CSS, but you want to intercede on the unrecognized things and fix it up into valid CSS.  (Probably don't want to do this - it collides with the custom property/function/at-rule case, but without the friendliness to future language extensions.)
2. Custom properties that are more complex than we allow you to specify - something that takes "`<length> || <color>`", for example. Right now your only choice is to give it the grammar `"*"` which just gives you a string that you have to parse yourself.  Similar with custom functions (`--conic-gradient(...)`) and custom at-rules (`@--svg`) - these will *rarely* fit in our limited set of allowed grammars.
3. CSS-like languages that want to rely on CSS's generic syntax, like CAS or that one mapping language I forget the name of. These have nothing to do with CSS and will likely result in something other than a stylesheet: CAS turns into a series of querySelector() and setAttribute() calls; the mapping thing turns into some data structures specific to that application.
4. Extending HTML/SVG attributes that use a CSS-like syntax, like `<img sizes>`. If you wanted to add support for the `h` descriptor to `sizes`, you currently have to write your own full-feature CSS parser. (`sizes` is pretty complex; you shouldn't skimp.) Better to let the engine parse it as "generic CSS", then you can recognize the parts you need and do image selection yourself.

Example of the problem
======================

Here is an example of some JS code that is wanting to parse out various CSS
types and also seperate out the values from their units.

```

		function parseValues(value,propertyName) {
			// Trim value on the edges
			value = value.trim();

			// Normalize letter-casing
			value = value.toLowerCase();

			// Map colors to a standard value (eg: white, blue, yellow)
			if (isKeywordColor(value)) { return "<color-keyword>"; }

			value = value.replace(/[#][0-9a-fA-F]+/g, '#xxyyzz');

			// Escapce identifiers containing numbers
			var numbers = ['ZERO','ONE','TWO','THREE','FOUR','FIVE','SIX','SEVEN','EIGHT','NINE'];

			value = value.replace(
				/([_a-z][-_a-z]|[_a-df-z])[0-9]+[-_a-z0-9]*/g,
				s=>numbers.reduce(
					(m,nstr,nint)=>m.replace(RegExp(nint,'g'),nstr),
					s
				)
			);

			// Remove any digits eg: 55px -> px, 1.5 -> 0.0, 1 -> 0
			value = value.replace(/(?:[+]|[-]|)(?:(?:[0-9]+)(?:[.][0-9]+|)|(?:[.][0-9]+))(?:[e](?:[+]|[-]|)(?:[0-9]+))?(%|e[a-z]+|[a-df-z][a-z]*)/g, "$1");
			value = value.replace(/(?:[+]|[-]|)(?:[0-9]+)(?:[.][0-9]+)(?:[e](?:[+]|[-]|)(?:[0-9]+))?/g, " <float> ");
			value = value.replace(/(?:[+]|[-]|)(?:[.][0-9]+)(?:[e](?:[+]|[-]|)(?:[0-9]+))?/g, " <float> ");
			value = value.replace(/(?:[+]|[-]|)(?:[0-9]+)(?:[e](?:[+]|[-]|)(?:[0-9]+))/g, " <float> ");
			value = value.replace(/(?:[+]|[-]|)(?:[0-9]+)/g, " <int> ");

			// Unescapce identifiers containing numbers
			value = numbers.reduce(
				(m,nstr,nint)=>m.replace(RegExp(nstr,'g'),nint),
				value
			)

			// Remove quotes
			value = value.replace(/('|‘|’|")/g, "");

			//
			switch(propertyName) {
				case 'counter-increment':
				case 'counter-reset':
					// Anonymize the user identifier
					value = value.replace(/[-_a-zA-Z0-9]+/g,' <custom-ident> ');
					break;
				case 'grid':
				case 'grid-template':
				case 'grid-template-rows':
				case 'grid-template-columns':
				case 'grid-template-areas':
					// Anonymize line names
					value = value.replace(/\[[-_a-zA-Z0-9 ]+\]/g,' <line-names> ');
					break;
				case '--var':
					// Replace (...), {...} and [...]
					value = value.replace(/[(](?:[^()]+|[(](?:[^()]+|[(](?:[^()]+|[(](?:[^()]+|[(](?:[^()]*)[)])*[)])*[)])*[)])*[)]/g, " <parentheses-block> ");
					value = value.replace(/[(](?:[^()]+|[(](?:[^()]+|[(](?:[^()]+|[(](?:[^()]+|[(](?:[^()]*)[)])*[)])*[)])*[)])*[)]/g, " <parentheses-block> ");
					value = value.replace(/\[(?:[^()]+|\[(?:[^()]+|\[(?:[^()]+|\[(?:[^()]+|\[(?:[^()]*)\])*\])*\])*\])*\]/g, " <curly-brackets-block> ");
					value = value.replace(/\[(?:[^()]+|\[(?:[^()]+|\[(?:[^()]+|\[(?:[^()]+|\[(?:[^()]*)\])*\])*\])*\])*\]/g, " <curly-brackets-block> ");
					value = value.replace(/\{(?:[^()]+|\{(?:[^()]+|\{(?:[^()]+|\{(?:[^()]+|\{(?:[^()]*)\})*\})*\})*\})*\}/g, " <square-brackets-block> ");
					value = value.replace(/\{(?:[^()]+|\{(?:[^()]+|\{(?:[^()]+|\{(?:[^()]+|\{(?:[^()]*)\})*\})*\})*\})*\}/g, " <square-brackets-block> ");
					break;
			}

			return value.trim();
		}
```

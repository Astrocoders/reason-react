---
title: Talk to Existing ReactJS Code
---

### Project Setup

You can reuse the _same_ bsb setup (that you might have seen [here](installation.md#bsb))! Aka, put a `bsconfig.json` at the root of your ReactJS project:

```json
{
  "name": "my-project-name",
  "reason": {"react-jsx" : 2},
  "sources": [
    "my_source_folder"
  ],
  "package-specs": [{
    "module": "commonjs",
    "in-source": true
  }],
  "suffix": ".bs.js",
  "namespace": true,
  "bs-dependencies": [
    "reason-react"
  ],
  "refmt": 3
}
```

This will build Reason files in `my_source_folder` (e.g. `reasonComponent.re`) and output the JS files (e.g. `reasonComponent.bs.js`) alongside them.

Then add `bs-platform` to your package.json (`npm install --save-dev bs-platform` or `yarn add --dev bs-platform`):

```json
"scripts": {
  "start": "bsb -make-world -w"
},
"devDependencies": {
  "bs-platform": "^2.1.0"
},
"dependencies": {
  "react": "^15.4.2",
  "react-dom": "^15.4.2",
  "reason-react": "^0.3.1"
}
...
```

Running `npm start` (or alias it to your favorite command) starts the `bsb` build watcher. **You don't have to touch your existing JavaScript build configuration**!

### Usage

A ReasonReact component **is not** a ReactJS component. We provide hooks to communicate between the two.

Whether you're using an existing ReactJS component or providing a ReasonReact component for consumption on the JS side, you need to establish the type of the JS props you'd convert from/to, by using [BuckleScript's `bs.deriving abstract`](https://bucklescript.github.io/docs/en/object.html):

```reason
[@bs.deriving abstract]
type jsProps = {
  /* some example fields */
  className: string,
  /* `type` is reserved in Reason. use `type_` and make it still compile to the
    JS key `type` */
  [@bs.as "type"] type_: string,
  value: Js.nullable(int),
};
```

This will generate the getters and the JS object creation function (of the same name, `jsProps`) you'll need.

**Note**: you do **not** declare `ref` and `key` (the two special ReactJS "props"). We handle that for you, just like ReactJS does. They're not really props.

#### ReasonReact using ReactJS

Easy! Since other Reason components only need you to expose a `make` function, fake one up:

```reason
[@bs.module] external myJSReactClass: ReasonReact.reactClass = "./myJSReactClass";

let make = (~className, ~type_, ~value=?, children) =>
  ReasonReact.wrapJsForReason(
    ~reactClass=myJSReactClass,
    ~props=jsProps(
      ~className,
      ~type_,
      ~value=Js.Nullable.fromOption(value),
    ),
    children,
  );
```

`ReasonReact.wrapJsForReason` is the helper we expose for this purpose. It takes in:

- The `reactClass` you want to wrap
- The `props` js object you'd create through the generated `jsProps` function from the `jsProps` type you've declared above (with values **properly converted** from Reason data structures to JS)
- The mandatory children you'd forward to the JS side.

`props` is mandatory. If you don't have any to pass, pass `~props=Js.Obj.empty()` instead.

**Note**: if your app successfully compiles, and you see the error "element type is invalid..." in your console, you might be hitting [this mistake](element-type-is-invalid.md).

### ReactJS Using ReasonReact

Eeeeasy. We expose a helper for the other direction, `ReasonReact.wrapReasonForJs`:

```reason
let component = ...;
let make ...;

[@bs.deriving abstract]
type jsProps = {
  name: string,
  age: Js.nullable(int),
};

let jsComponent =
  ReasonReact.wrapReasonForJs(~component, jsProps =>
    make(
      ~name=jsProps |. name,
      ~age=?Js.Nullable.toOption(jsProps |. age),
      [||],
    )
  );
```

The function takes in:

- The labeled reason `component` you've created
- A function that, given the JS props, asks you to call `make` while passing in the correctly converted parameters (`bs.deriving abstract` above generates a field accessor for every record field you've declared).

You'd assign the whole thing to the name `jsComponent`. The JS side can then import it:

```
var MyReasonComponent = require('./myReasonComponent.bs').jsComponent;
// make sure you're passing the correct data types!
<MyReasonComponent name="John" />
```

**Note**: if you'd rather use a **default import** on the JS side, you can export such default from BuckleScript/ReasonReact:

```reason
let default = ReasonReact.wrapReasonForJs(...)
```

and then import it on the JS side with:

```
import MyReasonComponent from './myReasonComponent.bs';
```

BuckleScript default exports **only** works when the JS side uses ES6 import/exports. [More info here](https://bucklescript.github.io/docs/en/import-export.html#export-an-es6-default-value).

---
order: 5
title: ESM
top_section: Filters
category: esm
---

The <a href="https://github.com/rubys/ruby2js/blob/master/lib/ruby2js/filter/esm.rb">esm</a> filter provides conversion of import and export statements for use with modern ES builders like Webpack.

## Examples

### import

```ruby
import "./index.scss"
# => import "./index.scss"

import Something from "./lib/something"
# => import Something from "./lib/something"

import Something, "./lib/something"
# => import Something from "./lib/something"

import [ LitElement, html, css ], from: "lit-element"
# => import { LitElement, html, css } from "lit-element"

import React, from: "react"
# => import React from "react"

import React, as: "*", from: "react"
# => import React as * from "react"
```

### export

```ruby
export hash = { ab: 123 }
# => export const hash = {ab: 123};

export func = ->(x) { x * 10 }
# => export const func = x => x * 10;

export def multiply(x, y)
  return x * y
end
# => export function multiply(x, y) {
#      return x * y
#    }

export default class MyClass
end
# => export default class MyClass {
#    };

# or final export statement:
export [ one, two, default: three ]
# => export { one, two, three as default }
```

If the "autoexports" option is `true`, all top level modules, classes, 
methods and constants will automatically be exported.

## Autoimports

The esm filter also provides a way to specify "autoimports" when you run the
conversion. It will add the relevant import statements automatically whenever
a particular class or function name is referenced. These can be either default
or named exports. Simply provide an `autoimports` hash with one or more keys
to the `Ruby2JS.convert` method. Examples:

```ruby
require "ruby2js/filter/esm"
puts Ruby2JS.convert('class MyElement < LitElement; end',
  eslevel: 2020, autoimports: {[:LitElement] => "lit-element"})
```

```js
// JavaScript output:
import { LitElement } from "lit-element"
class MyElement extends LitElement {}
```

```ruby
require "ruby2js/filter/esm"
puts Ruby2JS.convert('AWN.new({position: "top-right"}).success("Hello World")',
  eslevel: 2020, autoimports: {:AWN => "awesome-notifications"})
```

```js
// JavaScript output:
import AWN from "awesome-notifications"
new AWN({position: "top-right"}).success("Hello World")
```

The esm filter is able to recognize if you are defining a class or function
within the code itself and it won't add that import statement accordingly.
If for some reason you wish to disable autoimports entirely on a file-by-file
basis (for instance when using the Webpack loader), you can add a magic comment
to the top of the code:

```ruby
require "ruby2js/filter/esm"
puts Ruby2JS.convert(
  "# autoimports: false\n" +
  'AWN.new({position: "top-right"}).success("Hello World")',
  eslevel: 2020, autoimports: {:AWN => "awesome-notifications"}
)
```

```js
// autoimports: false
new AWN({position: "top-right"}).success("Hello World")
```
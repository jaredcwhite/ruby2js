Ruby2js
=======

Minimal yet extensible Ruby to JavaScript conversion.  

[![Build Status](https://travis-ci.org/rubys/ruby2js.svg)](https://travis-ci.org/rubys/ruby2js)
[![Gem Version](https://badge.fury.io/rb/ruby2js.svg)](https://badge.fury.io/rb/ruby2js)
[![Gitter](https://badges.gitter.im/ruby2js/community.svg)](https://gitter.im/ruby2js/community?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge)

Description
---

The base package maps Ruby syntax to JavaScript semantics.
For example:

  * a Ruby Hash literal becomes a JavaScript Object literal
  * Ruby symbols become JavaScript strings.
  * Ruby method calls become JavaScript function calls IF
    there are either one or more arguments passed OR
    parenthesis are used
  * otherwise Ruby method calls become JavaScript property accesses.
  * by default, methods and procs return `undefined`
  * splats mapped to spread syntax when ES2015 or later is selected, and
    to equivalents using `apply`, `concat`, `slice`, and `arguments` otherwise.
  * ruby string interpolation is expanded into string + operations
  * `and` and `or` become `&&` and `||`
  * `a ** b` becomes `Math.pow(a,b)`
  * `<< a` becomes `.push(a)`
  * `unless` becomes `if !`
  * `until` becomes `while !`
  * `case` and `when` becomes `switch` and `case`
  * ruby for loops become js for loops
  * `(1...4).step(2){` becomes `for (var i = 1; i < 4; i += 2) {`
  * `x.forEach { next }` becomes `x.forEach(function() {return})`
  * `lambda {}` and `proc {}` becomes `function() {}`
  * `class Person; end` becomes `function Person() {}`
  * instance methods become prototype methods
  * instance variables become underscored, `@name` becomes `this._name`
  * self is assigned to this is if used
  * Any block becomes and explicit argument `new Promise do; y(); end` becomes `new Promise(function() {y()})`
  * regular expressions are mapped to js
  * `raise` becomes `throw`

Ruby attribute accessors, methods defined with no parameters and no
parenthesis, as well as setter method definitions, are
mapped to
[Object.defineProperty](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/defineProperty?redirectlocale=en-US&redirectslug=JavaScript%2FReference%2FGlobal_Objects%2FObject%2FdefineProperty),
so avoid these if you wish to target users running IE8 or lower.

While both Ruby and JavaScript have open classes, Ruby unifies the syntax for
defining and extending an existing class, whereas JavaScript does not.  This
means that Ruby2JS needs to be told when a class is being extended, which is
done by prepending the `class` keyword with two plus signs, thus:
`++class C; ...; end`.

Filters may be provided to add Ruby-specific or framework specific
behavior.  Filters are essentially macro facilities that operate on
an AST representation of the code.

See
[notimplemented_spec](https://github.com/rubys/ruby2js/blob/master/spec/notimplemented_spec.rb)
for a list of Ruby features _known_ to be not implemented.

Demo quick start
---

The following two commands will start a server and a launch a browser:

```
git clone https://github.com/rubys/ruby2js.git
ruby ruby2js/demo/ruby2js.rb --port 8080
```

From the page that is loaded, enter some Ruby code into the text area
and press the convert button.  Dropdowns are provided to change the ECMAScript
level, filters, and options.  A checkbox is provided to show the Abstract
Symbol Tree (AST) produced.


Synopsis
---

Basic:

```ruby
require 'ruby2js'
puts Ruby2JS.convert("a={age:3}\na.age+=1")
```

With filter:

```ruby
require 'ruby2js/filter/functions'
puts Ruby2JS.convert('"2A".to_i(16)')
```

Host variable substitution:

```ruby
 puts Ruby2JS.convert("@name", ivars: {:@name => "Joe"})
```

Host expression evalution -- potentially unsafe, use only if you trust
the source being converted::
```ruby
 i = 7; puts Ruby2JS.convert("i = `i`", binding: binding)
```

Enable ES2015 support:

```ruby
puts Ruby2JS.convert('"#{a}"', eslevel: 2015)
```

Enable strict support:

```ruby
puts Ruby2JS.convert('a=1', strict: true)
```

Emit strict equality comparisons:

```ruby
puts Ruby2JS.convert('a==1', comparison: :identity)
```

Emit nullish coalescing operators:

```ruby
puts Ruby2JS.convert('a || 1', or: :nullish)
```

Emit underscored private fields (allowing subclass access):

```ruby
puts Ruby2JS.convert('class C; def initialize; @f=1; end; end',
  eslevel: 2020, underscored_private: true)
```

With [ExecJS](https://github.com/sstephenson/execjs):
```ruby
require 'ruby2js/execjs'
require 'date'

context = Ruby2JS.compile(Date.today.strftime('d = new Date(%Y, %-m-1, %-d)'))
puts context.eval('d.getYear()')+1900
```

Conversions can be explored interactively using the
[demo](https://github.com/rubys/ruby2js/blob/master/demo/ruby2js.rb) provided.

Introduction
---

JavaScript is a language where `0` is considered `false`, strings are
immutable, and the behaviors for operators like `==` are, at best,
[convoluted](https://zero.milosz.ca/).

Any attempt to bridge the semantics of Ruby and JavaScript will involve
trade-offs.  Consider the following expression:

```ruby
a[-1]
```

Programmers who are familiar with Ruby will recognize that this returns the
last element (or character) of an array (or string).  However, the meaning is
quite different if `a` is a Hash.

One way to resolve this is to change the way indexing operators are evaluated,
and to provide a runtime library that adds properties to global JavaScript
objects to handle this.  This is the approach that [Opal](https://opalrb.com/)
takes.  It is a fine approach, with a number of benefits.  It also has some
notable drawbacks.  For example,
[readability](https://opalrb.com/try/#code:a%20%3D%20%22abc%22%3B%20puts%20a[-1])
and
[compatibility with other frameworks](https://github.com/opal/opal/issues/400).

Another approach is to simply accept JavaScript semantics for what they are.
This would mean that negative indexes would return `undefined` for arrays
and strings.  This is the base approach provided by ruby2js.

A third approach would be to do static transformations on the source in order
to address common usage patterns or idioms.  These transformations can even be
occasionally unsafe, as long as the transformations themselves are opt-in.
ruby2js provides a number of such filters, including one that handles negative
indexes when passed as a literal.  As indicated above, this is unsafe in that
it will do the wrong thing when it encounters a hash index which is expressed
as a literal constant negative one.  My experience is that such is rare enough
to be safely ignored, but YMMV.  More troublesome, this also won’t work when
the index is not a literal (e.g., `a[n]`) and the index happens to be
negative at runtime.

This quickly gets into gray areas.  `each` in Ruby is a common method that
facilitates iteration over arrays.  `forEach` is the JavaScript equivalent.
Mapping this is fine until you start using a framework like jQuery which
provides a function named [each](https://api.jquery.com/jQuery.each/).

Fortunately, Ruby provides `?` and `!` as legal suffixes for method names,
Ruby2js filters do an exact match, so if you select a filter that maps `each`
to `forEach`, `each!` will pass through the filter.  The final code that emits
JavaScript function calls and parameter accesses will strip off these
suffixes.

This approach works well if it is an occasional change, but if the usage is
pervasive, most filters support options to `exclude` a list of mappings,
for example:

```ruby
puts Ruby2JS.convert('jQuery("li").each {|index| ...}', exclude: :each)
```

Alternatively, you can change the default:

```ruby
Ruby2JS::Filter.exclude :each
```

Static transformations and runtime libraries aren't aren’t mutually exclusive.
With enough of each, one could reproduce any functionality desired.

Integrations
---

While this is a low level library suitable for DIY integration, one of the
obvious uses of a tool that produces JavaScript is by web servers.  Ruby2JS
includes several integrations:

*  [CGI](https://github.com/rubys/ruby2js/blob/master/lib/ruby2js/cgi.rb)
*  [Sinatra](https://github.com/rubys/ruby2js/blob/master/lib/ruby2js/sinatra.rb)
*  [Rails](https://github.com/rubys/ruby2js/blob/master/lib/ruby2js/rails.rb)
*  [Haml](https://github.com/rubys/ruby2js/blob/master/lib/ruby2js/haml.rb)

As you might expect, CGI is a bit sluggish.  By contrast, Sinatra and Rails
are quite speedy as the bulk of the time is spent on the initial load of the
required libraries.

For easy integration with Webpack (and Webpacker in Rails 5+), you can use the
[rb2js-loader](https://github.com/whitefusionhq/rb2js-loader) plugin.

Filters
---

In general, making use of a filter is as simple as requiring it.  If multiple
filters are selected, they will all be applied in parallel in one pass through
the script.

* <a id="return" href="https://github.com/rubys/ruby2js/blob/master/lib/ruby2js/filter/return.rb">return</a>
  adds `return` to the last expression in functions.

* <a id="require" href="https://github.com/rubys/ruby2js/blob/master/lib/ruby2js/filter/require.rb">require</a>
  supports `require` and `require_relative` statements.  Contents of files
  that are required are converted to JavaScript and expanded inline.
  `require` function calls in expressions are left alone.

* <a id="camelCase" href="https://github.com/rubys/ruby2js/blob/master/lib/ruby2js/filter/camelCase.rb">camelCase</a>
  converts `underscore_case` to `camelCase`.  See
  [camelcase_spec](https://github.com/rubys/ruby2js/blob/master/spec/camelcase_spec.rb)
  for examples.

* <a id="functions" href="https://github.com/rubys/ruby2js/blob/master/lib/ruby2js/filter/functions.rb">functions</a>

    * `.abs` becomes `Math.abs()`
    * `.all?` becomes `.every`
    * `.any?` becomes `.some`
    * `.ceil` becomes `Math.ceil()`
    * `.chr` becomes `fromCharCode`
    * `.clear` becomes `.length = 0`
    * `.delete` becomes `delete target[arg]`
    * `.downcase` becomes `.toLowerCase`
    * `.each` becomes `.forEach`
    * `.each_key` becomes `for (i in ...) {}`
    * `.each_pair` becomes `for (var key in item) {var value = item[key]; ...}`
    * `.each_value` becomes `.forEach`
    * `.each_with_index` becomes `.forEach`
    * `.end_with?` becomes `.slice(-arg.length) == arg`
    * `.empty?` becomes `.length == 0`
    * `.find_index` becomes `findIndex`
    * `.first` becomes `[0]`
    * `.first(n)` becomes `.slice(0, n)`
    * `.floor` becomes `Math.floor()`
    * `.gsub` becomes `replace(//g)`
    * `.include?` becomes `.indexOf() != -1`
    * `.inspect` becomes `JSON.stringify()`
    * `.keys()` becomes `Object.keys()`
    * `.last` becomes `[*.length-1]`
    * `.last(n)` becomes `.slice(*.length-1, *.length)`
    * `.lstrip` becomes `.replace(/^\s+/, "")`
    * `.max` becomes `Math.max.apply(Math)`
    * `.merge` becomes `Object.assign({}, ...)`
    * `.merge!` becomes `Object.assign()`
    * `.min` becomes `Math.min.apply(Math)`
    * `.nil?` becomes `== null`
    * `.ord` becomes `charCodeAt(0)`
    * `puts` becomes `console.log`
    * `.replace` becomes `.length = 0; ...push.apply(*)`
    * `.respond_to?` becomes `right in left`
    * `.rstrip` becomes `.replace(/s+$/, "")`
    * `.scan` becomes `.match(//g)`
    * `.sum` becomes `.reduce(function(a, b) {a + b}, 0)`
    * `.start_with?` becomes `.substring(0, arg.length) == arg`
    * `.upto(lim)` becomes `for (var i=num; i<=lim; i+=1)`
    * `.downto(lim)` becomes `for (var i=num; i>=lim; i-=1)`
    * `.step(lim, n).each` becomes `for (var i=num; i<=lim; i+=n)`
    * `.step(lim, -n).each` becomes `for (var i=num; i>=lim; i-=n)`
    * `(0..a).to_a` becomes `Array.apply(null, {length: a}).map(Function.call, Number)`
    * `(b..a).to_a` becomes `Array.apply(null, {length: (a-b+1)}).map(Function.call, Number).map(function (idx) { return idx+b })`
    * `(b...a).to_a` becomes `Array.apply(null, {length: (a-b)}).map(Function.call, Number).map(function (idx) { return idx+b })`
    * `.strip` becomes `.trim`
    * `.sub` becomes `.replace`
    * `.tap {|n| n}` becomes `(function(n) {n; return n})(...)`
    * `.to_f` becomes `parseFloat`
    * `.to_i` becomes `parseInt`
    * `.to_s` becomes `.to_String`
    * `.upcase` becomes `.toUpperCase`
    * `.yield_self {|n| n}` becomes `(function(n) {return n})(...)`
    * `[-n]` becomes `[*.length-n]` for literal values of `n`
    * `[n...m]` becomes `.slice(n,m)`
    * `[n..m]` becomes `.slice(n,m+1)`
    * `[/r/, n]` becomes `.match(/r/)[n]`
    * `[/r/, n]=` becomes `.replace(/r/, ...)`
    * `(1..2).each {|i| ...}` becomes `for (var i=1 i<=2; i+=1)`
    * `"string" * length` becomes `new Array(length + 1).join("string")`
    * `.sub!` and `.gsub!` become equivalent `x = x.replace` statements
    * `.map!`, `.reverse!`, and `.select` become equivalent
      `.splice(0, .length, *.method())` statements
    * `@foo.call(args)` becomes `this._foo(args)`
    * `@@foo.call(args)` becomes `this.constructor._foo(args)`
    * `Array(x)` becomes `Array.prototype.slice.call(x)`
    * `delete x` becomes `delete x` (note lack of parenthesis)
    * `setInterval` and `setTimeout` allow block to be treated as the
       first parameter on the call
    * for the following methods, if the block consists entirely of a simple
      expression (or ends with one), a `return` is added prior to the
      expression: `sub`, `gsub`, `any?`, `all?`, `map`, `find`, `find_index`.
    * New classes subclassed off of `Exception` will become subclassed off
      of `Error` instead; and default constructors will be provided
    * `loop do...end` will be replaced with `while (true) {...}`
    * `raise Exception.new(...)` will be replaced with `throw new Error(...)`
    * `block_given?` will check for the presence of optional argument `_implicitBlockYield` which is a function made accessible through the use of `yield` in a method body.

    Additionally, there is one mapping that will only be done if explicitly
    included (pass `include: :class` as a `convert` option to enable):

    * `.class` becomes `.constructor`

* <a id="active_functions" href="https://github.com/rubys/ruby2js/blob/master/lib/ruby2js/filter/active_functions.rb">active_functions</a>

    Provides functionality inspired by Rails' ActiveSupport. Works in conjunction
    with the tiny NPM dependency `@ruby2js/active-functions` which must be added to
    your application.

    * `value.blank?` becomes `blank$(value)`
    * `value.present?` becomes `present$(value)`
    * `value.presence` becomes `presence$(value)`

    Note: these conversions are only done if eslevel >= 2015. Import statements
    will be added to the top of the code output automatically. By default they
    will be `@ruby2js/active-functions`, but you can pass an `import_from_skypack: true` option to `convert` to use the Skypack CDN instead.

* <a id="tagged_templates" href="https://github.com/rubys/ruby2js/blob/master/lib/ruby2js/filter/tagged_templates.rb">tagged_templates</a>

    Allows you to turn certain method calls with a string argument into tagged
    template literals. By default it supports html and css, so you can write
    `html "<div>#{1+2}</div>"` which converts to `` html`<div>${1+2}</div>` ``.
    Works nicely with squiggly heredocs for multi-line templates as well. If you
    need to configure the tag names yourself, pass a `template_literal_tags`
    option to `convert` with an array of tag name symbols.

    Note: these conversions are only done if eslevel >= 2015

* <a href="https://github.com/rubys/ruby2js/blob/master/lib/ruby2js/filter/esm.rb">esm</a>

    Provides conversion of import and export statements for use with modern ES builders like Webpack.

    Examples:

    **import**

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

    **export**

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

* <a id="node" href="https://github.com/rubys/ruby2js/blob/master/spec/node_spec.rb">node</a>

    * `` `command` `` becomes `child_process.execSync("command", {encoding: "utf8"})`
    * `ARGV` becomes `process.argv.slice(2)`
    * `__dir__` becomes `__dirname`
    * `Dir.chdir` becomes `process.chdir`
    * `Dir.entries` becomes `fs.readdirSync`
    * `Dir.home` becomes `os.homedir()`
    * `Dir.mkdir` becomes `fs.mkdirSync`
    * `Dir.mktmpdir` becomes `fs.mkdtempSync`
    * `Dir.pwd` becomes `process.cwd`
    * `Dir.rmdir` becomes `fs.rmdirSync`
    * `Dir.tmpdir` becomes `os.tmpdir()`
    * `ENV` becomes `process.env`
    * `__FILE__` becomes `__filename`
    * `File.absolute_path` becomes `path.resolve`
    * `File.absolute_path?` becomes `path.isAbsolute`
    * `File.basename` becomes `path.basename`
    * `File.chmod` becomes `fs.chmodSync`
    * `File.chown` becomes `fs.chownSync`
    * `File.cp` becomes `fs.copyFileSync`
    * `File.dirname` becomes `path.dirname`
    * `File.exist?` becomes `fs.existsSync`
    * `File.extname` becomes `path.extname`
    * `File.join` becomes `path.join`
    * `File.lchmod` becomes `fs.lchmodSync`
    * `File.link` becomes `fs.linkSync`
    * `File.ln` becomes `fs.linkSync`
    * `File.lstat` becomes `fs.lstatSync`
    * `File::PATH_SEPARATOR` becomes `path.delimiter`
    * `File.read` becomes `fs.readFileSync`
    * `File.readlink` becomes `fs.readlinkSync`
    * `File.realpath` becomes `fs.realpathSync`
    * `File.rename` becomes `fs.renameSync`
    * `File::SEPARATOR` becomes `path.sep`
    * `File.stat` becomes `fs.statSync`
    * `File.symlink` becomes `fs.symlinkSync`
    * `File.truncate` becomes `fs.truncateSync`
    * `File.unlink` becomes `fs.unlinkSync`
    * `FileUtils.cd` becomes `process.chdir`
    * `FileUtils.cp` becomes `fs.copyFileSync`
    * `FileUtils.ln` becomes `fs.linkSync`
    * `FileUtils.ln_s` becomes `fs.symlinkSync`
    * `FileUtils.mkdir` becomes `fs.mkdirSync`
    * `FileUtils.mv` becomes `fs.renameSync`
    * `FileUtils.pwd` becomes `process.cwd`
    * `FileUtils.rm` becomes `fs.unlinkSync`
    * `IO.read` becomes `fs.readFileSync`
    * `IO.write` becomes `fs.writeFileSync`
    * `system` becomes `child_process.execSync(..., {stdio: "inherit"})`

* <a id="nokogiri" href="https://github.com/rubys/ruby2js/blob/master/spec/nokogiri.rb">nokogiri</a>
    * `add_child` becomes `appendChild`
    * `add_next_sibling` becomes `node.parentNode.insertBefore(sibling, node.nextSibling)`
    * `add_previous_sibling` becomes `node.parentNode.insertBefore(sibling, node)`
    * `after` becomes `node.parentNode.insertBefore(sibling, node.nextSibling)`
    * `at` becomes `querySelector`
    * `attr` becomes `getAttribute`
    * `attribute` becomes `getAttributeNode`
    * `before` becomes `node.parentNode.insertBefore(sibling, node)`
    * `cdata?` becomes `node.nodeType === Node.CDATA_SECTION_NODE`
    * `children` becomes `childNodes`
    * `comment?` becomes `node.nodeType === Node.COMMENT_NODE`
    * `content` becomes `textContent`
    * `create_element` becomes `createElement`
    * `document` becomes `ownerDocument`
    * `element?` becomes `node.nodeType === Node.ELEMENT_NODE`
    * `fragment?` becomes `node.nodeType === Node.FRAGMENT_NODE`
    * `get_attribute` becomes `getAttribute`
    * `has_attribute` becomes `hasAttribute`
    * `inner_html` becomes `innerHTML`
    * `key?` becomes `hasAttribute`
    * `name` becomes `nextSibling`
    * `next` becomes `nodeName`
    * `next=` becomes `node.parentNode.insertBefore(sibling,node.nextSibling)`
    * `next_element` becomes `nextElement`
    * `next_sibling` becomes `nextSibling`
    * `Nokogiri::HTML5` becomes `new JSDOM().window.document`
    * `Nokogiri::HTML5.parse` becomes `new JSDOM().window.document`
    * `Nokogiri::HTML` becomes `new JSDOM().window.document`
    * `Nokogiri::HTML.parse` becomes `new JSDOM().window.document`
    * `Nokogiri::XML::Node.new` becomes `document.createElement()`
    * `parent` becomes `parentNode`
    * `previous=` becomes `node.parentNode.insertBefore(sibling, node)`
    * `previous_element` becomes `previousElement`
    * `previous_sibling` becomes `previousSibling`
    * `processing_instruction?` becomes `node.nodeType === Node.PROCESSING_INSTRUCTION_NODE`
    * `remove_attribute` becomes `removeAttribute`
    * `root` becomes `documentElement`
    * `search` becomes `querySelectorAll`
    * `set_attribute` becomes `setAttribute`
    * `text?` becomes `node.nodeType === Node.TEXT_NODE`
    * `text` becomes `textContent`
    * `to_html` becomes `outerHTML`

* <a id="underscore" href="https://github.com/rubys/ruby2js/blob/master/spec/underscore.rb">underscore</a>

    * `.clone()` becomes `_.clone()`
    * `.compact()` becomes `_.compact()`
    * `.count_by {}` becomes `_.countBy {}`
    * `.find {}` becomes `_.find {}`
    * `.find_by()` becomes `_.findWhere()`
    * `.flatten()` becomes `_.flatten()`
    * `.group_by {}` becomes `_.groupBy {}`
    * `.has_key?()` becomes `_.has()`
    * `.index_by {}` becomes `_.indexBy {}`
    * `.invert()` becomes `_.invert()`
    * `.invoke(&:n)` becomes `_.invoke(, :n)`
    * `.map(&:n)` becomes `_.pluck(, :n)`
    * `.merge!()` becomes `_.extend()`
    * `.merge()` becomes `_.extend({}, )`
    * `.reduce {}` becomes `_.reduce {}`
    * `.reduce()` becomes `_.reduce()`
    * `.reject {}` becomes `_.reject {}`
    * `.sample()` becomes `_.sample()`
    * `.select {}` becomes `_.select {}`
    * `.shuffle()` becomes `_.shuffle()`
    * `.size()` becomes `_.size()`
    * `.sort()` becomes `_.sort_by(, _.identity)`
    * `.sort_by {}` becomes `_.sortBy {}`
    * `.times {}` becomes `_.times {}`
    * `.values()` becomes `_.values()`
    * `.where()` becomes `_.where()`
    * `.zip()` becomes `_.zip()`
    * `(n...m)` becomes `_.range(n, m)`
    * `(n..m)` becomes `_.range(n, m+1)`
    * `.compact!`, `.flatten!`, `shuffle!`, `reject!`, `sort_by!`, and
      `.uniq` become equivalent `.splice(0, .length, *.method())` statements
    * for the following methods, if the block consists entirely of a simple
      expression (or ends with one), a `return` is added prior to the
      expression: `reduce`, `sort_by`, `group_by`, `index_by`, `count_by`,
      `find`, `select`, `reject`.
    * `is_a?` and `kind_of?` map to `Object.prototype.toString.call() ===
      "[object #{type}]" for the following types: `Arguments`, `Boolean`,
      `Date`, `Error`, `Function`, `Number`, `Object`, `RegExp`, `String`; and
      maps Ruby names to JavaScript equivalents for `Exception`, `Float`,
      `Hash`, `Proc`, and `Regexp`.  Additionally, `is_a?` and `kind_of?` map
      to `Array.isArray()` for `Array`.

* <a id="jquery" href="https://github.com/rubys/ruby2js/blob/master/spec/jquery.rb">jquery</a>

    * maps Ruby unary operator `~` to jQuery `$` function
    * maps Ruby attribute syntax to jquery attribute syntax
    * `.to_a` becomes `toArray`
    * maps `$$` to jQuery `$` function
    * defaults the fourth parameter of $$.post to `"json"`, allowing Ruby block
      syntax to be used for the success function.

* <a id="minitest-jasmine" href="https://github.com/rubys/ruby2js/blob/master/spec/minitest-jasmine.rb">minitest-jasmine</a>
    * maps subclasses of `Minitest::Test` to `describe` calls
    * maps `test_` methods inside subclasses of `Minitest::Test` to `it` calls
    * maps `setup`, `teardown`, `before`, and `after` calls to `beforeEach`
      and `afterEach` calls
    * maps `assert` and `refute` calls to `expect`...`toBeTruthy()` and
      `toBeFalsy` calls
    * maps `assert_equal`, `refute_equal`, `.must_equal` and `.cant_equal`
      calls to `expect`...`toBe()` calls
    * maps `assert_in_delta`, `refute_in_delta`, `.must_be_within_delta`,
      `.must_be_close_to`, `.cant_be_within_delta`, and `.cant_be_close_to`
      calls to `expect`...`toBeCloseTo()` calls
    * maps `assert_includes`, `refute_includes`, `.must_include`, and
      `.cant_include` calls to `expect`...`toContain()` calls
    * maps `assert_match`, `refute_match`, `.must_match`, and `.cant_match`
      calls to `expect`...`toMatch()` calls
    * maps `assert_nil`, `refute_nil`, `.must_be_nil`, and `.cant_be_nill` calls
      to `expect`...`toBeNull()` calls
    * maps `assert_operator`, `refute_operator`, `.must_be`, and `.cant_be`
       calls to `expect`...`toBeGreaterThan()` or `toBeLessThan` calls

* <a id="cjs" href="https://github.com/rubys/ruby2js/blob/master/spec/cjs">cjs</a>
    * maps `export def f` to `exports.f =`
    * maps `export async def f` to `exports.f = async`
    * maps `export v =` to `exports.v =`
    * maps `export default proc` to `module.exports =`
    * maps `export default async proc` to `module.exports = async`
    * maps `export default` to `module.exports =`

* <a id="matchAll" href="https://github.com/rubys/ruby2js/blob/master/spec/matchAll">matchAll</a>

    For ES level < 2020:

    * maps `str.matchAll(pattern).forEach {}` to 
      `while (match = pattern.exec(str)) {}`

    Note `pattern` must be a simple variable with a value of a regular
    expression with the `g` flag set at runtime.

[Wunderbar](https://github.com/rubys/wunderbar) includes additional demos:

* [chat](https://github.com/rubys/wunderbar/blob/master/demo/chat.rb),
  [diskusage](https://github.com/rubys/wunderbar/blob/master/demo/diskusage.rb),
  and [wiki](https://github.com/rubys/wunderbar/blob/master/demo/wiki.rb) make
  use of the jquery filter.

ES2015 support
---

When option `eslevel: 2015` is provided, the following additional
conversions are made:

* `"#{a}"` becomes <code>\`${a}\`</code>
* `a = 1` becomes `let a = 1`
* `A = 1` becomes `const A = 1`
* `a, b = b, a` becomes `[a, b] = [b, a]`
* `a, (foo, *bar) = x` becomes `let [a, [foo, ...bar]] = x`
* `def f(a, (foo, *bar))` becomes `function f(a, [foo, ...bar])`
* `def a(b=1)` becomes `function a(b=1)`
* `def a(*b)` becomes `function a(...b)`
* `.each_value` becomes `for (i of ...) {}`
* `a(*b)` becomes `a(...b)`
* `"#{a}"` becomes <code>\`${a}\`</code>
* `lambda {|x| x}` becomes `(x) => {return x}`
* `proc {|x| x}` becomes `(x) => {x}`
* `a {|x|}` becomes `a((x) => {})`
* `class Person; end` becomes `class Person {}`
* `(0...a).to_a` becomes `[...Array(a).keys()]`
* `(0..a).to_a` becomes `[...Array(a+1).keys()]`
* `(b..a).to_a` becomes `Array.from({length: (a-b+1)}, (_, idx) => idx+b)`

ES2015 class support includes constructors, super, methods, class methods,
instance methods, instance variables, class variables, getters, setters,
attr_accessor, attr_reader, attr_writer, etc.

Additionally, the `functions` filter will provide the following conversion:

* `Array(x)` becomes `Array.from(x)`
* `.inject(n) {}` becomes `.reduce(() => {}, n)`

Keyword arguments and optional keyword arguments will be mapped to
parameter destructuring.

Classes defined with a `method_missing` method will emit a `Proxy` object
for each instance that will forward calls.  Note that in order to forward
arguments, this proxy will return a function that will need to be called,
making it impossible to proxy attributes/getters.  As a special accommodation,
if the `method_missing` method is defined to only accept a single parameter
it will be called with only the method name, and it is free to return
either values or functions.

ES2016 support
---

When option `eslevel: 2016` is provided, the following additional
conversion is made:

* `a ** b` becomes `a ** b`

Additionally the following conversions is added to the `functions` filter:

* `.include?` becomes `.includes`

ES2017 support
---

When option `eslevel: 2017` is provided, the following additional
conversions are made by the `functions` filter:

* `.values()` becomes `Object.values()`
* `.entries()` becomes `Object.entries()`
* `.each_pair {}` becomes `for (let [key, value] of Object.entries()) {}`

async support:

* `async def` becomes `async function`
* `async lambda` becomes `async =>`
* `async proc` becomes `async =>`
* `async ->` becomes `async =>`
* `foo bar, async do...end` becomes `foo(bar, async () => {})`

ES2018 support
---

When option `eslevel: 2018` is provided, the following additional
conversion is made by the `functions` filter:

* `.merge` becomes `{...a, ...b}`

Additionally, rest arguments can now be used with keyword arguments and
optional keyword arguments.

ES2019 support
---

When option `eslevel: 2019` is provided, the following additional
conversion is made by the `functions` filter:

* `.flatten` becomes `.flat(Infinity)`
* `.lstrip` becomes `.trimEnd`
* `.rstrip` becomes `.trimStart`
* `a.to_h` becomes `Object.fromEntries(a)`
* `Hash[a]` becomes `Object.fromEntries(a)`

Additionally, `rescue` without a variable will map to `catch` without a
variable.

ES2020 support
---

When option `eslevel: 2020` is provided, the following additional
conversions are made:

* `@x` becomes `this.#x`
* `@@x` becomes `ClassName.#x`
* `a&.b` becomes `a?.b`
* `.scan` becomes `Array.from(str.matchAll(/.../g), s => s.slice(1))`

ES2021 support
---

When option `eslevel: 2021` is provided, the following additional
conversions are made:

* `x ||= 1` becomes `x ||= 1`
* `x &&= 1` becomes `x &&= 1`
* `1000000.000001` becomes `1_000_000.000_001`

Picking a Ruby to JS mapping tool
---

> dsl — A domain specific language, where code is written in one language and
> errors are given in another.
> -- [Devil’s Dictionary of Programming](https://programmingisterrible.com/post/65781074112/devils-dictionary-of-programming)

If you simply want to get a job done, and would like a mature and tested
framework, and only use one of the many integrations that
[Opal](https://opalrb.com/) provides, then Opal is the way to go right now.

ruby2js is for those that want to produce JavaScript that looks like it
wasn’t machine generated, and want the absolute bare minimum in terms of
limitations as to what JavaScript can be produced.

[Try](https://intertwingly.net/projects/ruby2js) for yourself.
[Compare](https://opalrb.com/try/#code:).

And, of course, the right solution might be to use
[CoffeeScript](https://coffeescript.org/) instead.

License
---

(The MIT License)

Copyright (c) 2009, 2020 Macario Ortega, Sam Ruby, Jared White

Permission is hereby granted, free of charge, to any person obtaining
a copy of this software and associated documentation files (the
'Software'), to deal in the Software without restriction, including
without limitation the rights to use, copy, modify, merge, publish,
distribute, sublicense, and/or sell copies of the Software, and to
permit persons to whom the Software is furnished to do so, subject to
the following conditions:

The above copyright notice and this permission notice shall be
included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED 'AS IS', WITHOUT WARRANTY OF ANY KIND,
EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF
MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT.
IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY
CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT,
TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE
SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.

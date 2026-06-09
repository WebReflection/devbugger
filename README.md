# DevBugger

A tree-shaking based approach to `D&&BUG` anything you like **only** when its state is `active` (*enabled* or simply `true`).

The *binary* creates or manages a tiny module you can statically *import* anywhere:

```js
import { D, BUG } from './src/devbugger.js';

D&&BUG.log('it works 🥳');
```

When `D` is `false`, the `D&&BUG...` expression is dead code that bundlers happily strip away, so nothing debugging-related ends up in your production bundle. Flip `D` to `true` and all your debug utilities are back, no source changes required.

## How to use the binary

```sh
# Interactive mode: creates the file if missing (after asking),
# then asks whether debugging should be enabled or disabled.
# Pressing Enter accepts the default, which is "yes".
npx devbugger ./src/devbugger.js

# works with bunx too
bunx devbugger ./src/devbugger.js

# Automatically switch it ON - no question asked.
# `--enable` is an alias of `--enabled`.
npx devbugger ./src/devbugger.js --enable # --enabled

# Automatically switch it OFF - no question asked.
# `--disable` is an alias of `--disabled`.
npx devbugger ./src/devbugger.js --disable # --disabled
```

### Behavior

- **File missing** — you are asked to create it at its resolved absolute path. Decline and the command exits cleanly without writing anything; accept and you are then asked whether debugging should be enabled.
- **File present** — you are asked whether to flip the `export const D` flag on or off; it is never overwritten otherwise.
- **`--enable` / `--disable`** — skip every question and apply the chosen state directly (creating the file first if needed). Passing both at once is an error and prints the help.
- **No arguments** — prints the help.

All prompts accept `y|Y|yes|YES|Yes|ok` and `n|N|no|NO|No|nope`, and default to *yes* on an empty answer.

## How does it work

This module provides a *pattern* rather than a whole utility.

The main concept is that via *tree-shaking* bundlers are able to avoid shipping unnecessary code, but when it's needed the code can be there, if convenient.

Looking at the previous example:

```js
import { D, BUG } from './src/devbugger.js';

D&&BUG.log('it works 🥳');
```

If the `D` flag is `false`, the produced output from that file will be something similar to:

```js
const o=!1,t=o;export{t as BUG,o as D};
```

This code should hint the *JS* engine there is dead code around, but most importantly, your final bundle size will be preserved.

On the other hand, if the `D` flag is `true`, the resulting code will be like:

```js
const o=!0,l={log:console.log};export{l as BUG,o as D};
```

That means the more named debugging details/utilities you'll add to that `./src/devbugger.js` file, the more handy logs you can read when your bundle builds your final project forcing `D` flag to be `true`.

**Please note** the utility only ever flips the `export const D` flag of an existing file; it never overwrites your code. A brand new file is created solely when it does not exist yet (and only after you confirm, or pass `--enable`/`--disable`).

## The exports contract

The two lines at the bottom of the file are the part that makes the whole pattern work, so **never change or remove them**:

```js
// ⚠️  DO NOT CHANGE THESE EXPORTS
export const D = true; // `false` to avoid bloat
export const BUG = D && bugger;
```

The only value the binary ever touches is `D` (`true` or `false`); everything else in those two lines must stay exactly as it is.

Both exports expect a `const bugger = {}` declaration **before** them. That `bugger` reference can be anything that exposes the debugging info you care about:

```js
// a namespace of helpers
const bugger = { log: console.log, table: console.table };

// or a plain callback
const bugger = (...args) => console.log('[debug]', ...args);

// or an instance of something
const bugger = new MyDebugger({ verbose: true });
```

Because `BUG` is `D && bugger`, when `D` is `false` the export collapses to `false` and every consuming expression becomes dead code. That is the whole point: calls like

```js
D&&BUG('hello', 'world');
D&&BUG.anything('at', 'all');
D&&BUG.toString();
```

are all completely ignored — and stripped by the bundler's tree-shaking — once `D` is `false`, no matter how `bugger` is shaped. Flip `D` back to `true` and the exact same expressions run against your real `bugger` again.

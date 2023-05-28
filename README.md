# Proposal: Clone Operator for JavaScript
* Unofficial, Incubation — not even T39 **Stage 0**

## What
Provide an operator (!) for shallow cloning any object.

```js
const duo = ['Laurel', 'Hardy'];
const stooges = ['Moe', 'Shemp', 'Larry', 'Curly', 'Corey'];
const lineup = stooges! //copy before mutation
  .slice(0, 2)
  .splice(0, 2)
  .reverse()
  .concat(duo);
```

## Why
FP has gained popularity because of the way it segregates programs into pure and impure modules.  One of its go-to practices is copy before mutation.  That is, if you want simulate state change with its native objects and arrays you can, as long as you take care to clone the object before mutating its copy.

Copy first is so prevalent it can be made ever more convenient.

### Faux Commands
The term *commands* is being used as functions with side effects, *queries* as functions without.

Arrays have several mutating commands: `sort`, `reverse`, `splice`.  But the advent of FP has caused devs to rethink how to simulate mutations using queries.

Recently, JavaScript arrays got `toSorted`, `toReversed` and `toSpliced`, what can be termed *faux commands*.  They're essentially the same as the original side-effecting commands except they copy first.

```js
  function toSorted(...args){
    return this.slice().sort(...args); //slice = copy
  }
```
A faux command is effectively a query which given some subject and a desired change returns a new copy of the subject with the change applied.  They're ordinarily used in meaningful programs with state containers, what Clojure calls atoms.

```js
const nums = [8, 6, 7, 5, 3, 0, 9];
const state = atom(nums);
state.swap(a => a.toSorted()); //nums is not mutated!
const modNums = state.deref(); //extract the replacement object
nums === modNums; //false
```
Functional programming simulates side effects and then later applies the fruit of the computation to achieve actual side effects—since a program without side effects does nothing.  It separates the pristine and pure from the messy and impure for greater good.

Faux commands are the bread and butter of this kind of separation.  They rely on simulation before effect.  Invariably, they're useful to have as evidenced by the eventual appearance of the `toSorted`, `toReversed` and `toSpliced` faux commands.  In practice, a faux command in its simplest form is a copy-before-mutation operation.

In its optimal form it may be used with (persistent) data types whose simulated commands benefit from efficient, under-the-hood structural reuse.  Clojure's go-to structures, maps and vectors, do precisely this.  Regardless of the type—object, array, map, or vector—reaching for faux commands allows one to simulate effects and solve problems functionally.

Although the above mentioned some development work surrounding arrays, that was only a single illustration.  **The same thinking can be applied to all native and custom types.** Invariably, the cornerstone of purity when you're working with nonpersistent data types like objects, arrays, and their otherwise mutable derivatives is clone.

Here's an example of an *impure* command...

```js
const hero = {name: "Robin Hood", weapon: "Bow", arrows: 0};

function resupply(archer){ //command
  return archer.increment("arrows", 25);
}
```
...which can be transformed into a pure faux command with a single character.  That's because clone is the necessary first operation in *any method chain or pipeline* when reference types are your staple constructs.

```js
function resupply(archer){ //faux command
  return archer!.increment("arrows", 25);
}
```
Clojure offers both kinds of commands.  It's just that, by default, since everything is immutable its primitives (e.g. maps and vectors) prefer faux commands (i.e. queries) to actual commands.  Change is, by necessity, simulated before it's applied.

But a series of chained faux commands is often costlier from a performance perspective than beginning with a clone operation and chaining a series of in-place mutations against a mutable type.  Simulating change adds an overhead that actual change does not.  That's why Clojure has transients.  They allow the programmer to temporarily opt out of simulated change to get performance.

```js
const grades = {A: 1, B: 2, C: 3, D: 4, F: 5};
function better(c1, c2){
  return grades[c1.grade] - grades[c2.grade];
}
const reportCards = [...]; //dozens
const honorRollCards = [...]; //dozens

//faux command method chain
const topTen = reportCards
  .toSpliced(0, 0, ...honorRollCards)
  .toSorted(better)
  .toReversed()
  .slice(0, 10);
```
While the above is contrived, it uses a functional approach to computing an outcome.  The inefficiency is hidden with each `toWhatever` call.  The reality is actually closer to this:

```js
const topTen = reportCards
  .slice() // clone
  .splice(0, 0, ...honorRollCards)
  .slice() // clone
  .sort(better)
  .slice() // clone
  .reverse()
  .slice(0, 10);
```
This demonstrates why faux commands (e.g. queries) are traded for actual commands.  The more intermediary operations which have a clone baked in the more pronounced the benefit of cloning and using regular commands becomes.  *This is JavaScript so the assumption is objects and arrays are being used rather than persistent data structures.*

Normally, copy-and-mutate method chains are wrapped in functions.  Although mutation occurs, it's hidden by the initial clone.  And the function is kept pure.

Also note a single initial clone is optimal.  There's no need for additional intermediary cloning.

```js
function getTopTen(reportCards, honorsReportCards){ //don't actually mutate these args!
  return reportCards
    .slice() //clone
    .splice(0, 0, ...honorsReportCards)
    .sort(better)
    .reverse()
    .slice(0, 10);
}
```
The clone operator provides a consistent means of beginning any such chain.

Taking the above one step further:
```js
function getTopTen(reportCards, honorsReportCards){ //don't actually mutate these args!
  return reportCards!
    .splice(0, 0, ...honorsReportCards)
    .sort(better)
    .reverse()
    .slice(0, 10);
}
```
It's because copy-first is so ubiquitously the first necessary ingredient to maintaining purity in programs which rely primarily on JavaScript primitives (arrays and object) and which do not otherwise want (or require) more sophisticated persistent structures that it makes sense to allow this to be kicked off in a consistent and obvious manner.

It abstracts aways the differences of the clone operation on arrays, objects, dates, etc. and binds them with a single common approach.

## How
Use a well-defined symbol `Symbol.clone` so that any object other than functions can supply a function which shallow clones itself and which gets invoked when the operator is applied.

This applies to both reference (object, array, date) and value types (record, tuple, number, string, etc.).  The difference is value types, being immutable, return themselves.  They still implement `clone` to maintain a common interface.

## Further Considerations
* This proposal complements the [command syntax proposal](https://github.com/mlanza/proposal-command-syntax).
* This should not be confused by the parser as a logical not (`!`) — `const restaurant = cash > 12, tvDinner = !restaurant`
* Clone operators don't work on functions due to that proposal.
* Deep cloning?  Separate `!!` operator?

## Examples
```js
const a = [8, 6, 7, 5, 3, 0, 9];
const b = a.toSorted().toReversed(); //less efficient
```
```js
const a = [8, 6, 7, 5, 3, 0, 9];
const b = a.slice().sort().reverse(); //more efficient
```
```js
const a = [8, 6, 7, 5, 3, 0, 9];
const b = a!.sort().reverse(); //better
```
```js
const a = [8, 6, 7, 5, 3, 0, 9];
const b = a!.sort!().reverse!(); //still better (feat. command syntax)
```

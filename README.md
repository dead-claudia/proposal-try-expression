# Try expressions

## Introduction

Catching and reifying an exception is a pretty common want and need:

- https://www.npmjs.com/package/nice-try
- https://www.npmjs.com/package/try-catch
- https://lodash.com/docs/4.17.15#attempt
- https://ramdajs.com/docs/#tryCatch
- https://github.com/petkaantonov/bluebird/blob/master/src/util.js#L30
- And countless others: https://www.npmjs.com/search?q=try%20catch
- V8's API exposes most failable actions via a `Maybe<T>` or `MaybeLocal<T>`, and its `TryCatch` API resembles this quite a bit.
- The ES spec does this *extensively* for basically every operation it has. Polyfills also do this to some extent because of this.

All of these boil down to one of a few variations of the same utility function:

```js
// tryCatch(func) -> [error, result]
function tryCatch(func) {
  try {
    return [null, func()]
  } catch (e) {
    return [e, null]
  }
}

// tryCatch(func) -> result | null
function tryCatch(func) {
  try {
    return func()
  } catch (e) {
    return null
  }
}

// tryCatch(func) -> result | error
function tryCatch(func) {
  try {
    return func()
  } catch (e) {
    return e
  }
}

// tryCatch(func) -> {caught, value}
function tryCatch(func) {
  try {
    return {caught: false, value: func()}
  } catch (e) {
    return {caught: true, value: e}
  }
}
```

I've seen all of these in the wild and personally have used all but the first in impromptu utilities on the fly. But this is an area where the engine could really help out massively - it could do this virtually zero-cost, and it could even make it work with `async`/`await` with basically no effort at all.

## Proposal

I propose we should create a new `try` expression. This would simplify a lot of common error handling tasks, and it'd synergize very well with the [pattern matching proposal](https://github.com/tc39/proposal-pattern-matching). It would operate basically like this:

```js
const result = try expr
```

This would desugar roughly to the following:

```js
let $result // This variable is purely internal
try {
  $result = {caught: false, value: expr}
} catch (e) {
  $result = {caught: true, error: e}
}

const result = $result
```

This is compatible with both `yield` and `await`, and such expressions can be anywhere. No restrictions are placed on the operand aside from it must be a valid unary expression and it can't start with a literal `{` (to prevent ambiguity). For example:

```js
const addResult = try await save(await transformData())
```

The above desugars to this:

```js
let $result // This variable is purely internal
try {
  $result = {caught: false, value: await save(await transformData())}
} catch (e) {
  $result = {caught: true, result: e}
}

const addResult = $result
```

### Spec

The grammar is super simple:

```
UnaryExpression ::
  `try` UnaryExpression
```

The semantics are also similarly simple:

`` UnaryExpression :: `try` UnaryExpression ``

1. Let *E* be the result of evaluating *UnaryExpression*.
2. If *E*.[[Type]] is **normal**, let *caught* be **`false`** and *valueKey* be **`"value"`**.
3. Else, if *E*.[[Type]] is **throw**, let *caught* be **`true`** and *valueKey* be **`"error"`**.
4. Else, return Completion(*E*).
5. Let *result* be ObjectCreate(%ObjectPrototype%).
6. Perform ! CreateDataProperty(*result*, **`"caught"`**, *caught*).
7. Perform ! CreateDataProperty(*result*, *valueKey*, *E*.[[Value]]).
8. Return NormalCompletion(*result*).

## Examples

Here, I've got two examples, one based on example code in a blog post and one based on code out in the wild.

### Performing an async task

Adapted from https://blog.grossman.io/how-to-write-async-await-without-try-catch-blocks-in-javascript/

```js
async function asyncTask() {
  const userReq = try await UserModel.findById(1)
  if (userReq.caught) throw new CustomerError('No user found')
  const user = userReq.value

  const taskReq = try await TaskModel({userId: user.id, name: 'Demo Task'})
  if (taskReq.caught) throw new CustomError('Error occurred while saving task', taskReq.error)

  if (user.notificationsEnabled) {
    const notificationReq = try await NotificationService.sendNotification(user.id, 'Task Created')
    if (notificationReq.caught) console.error('Error occurred while sending notification', notificationReq.error)
  }
}
```

Using `try`/`catch` instead:

```js
async function asyncTask() {
  let user

  try {
    user = await UserModel.findById(1)
  } catch {
    throw new CustomerError('No user found')
  }

  try {
    await TaskModel({userId: user.id, name: 'Demo Task'})
  } catch {
    throw new CustomError('Error occurred while saving task')
  }

  if (user.notificationsEnabled) {
    try {
      await NotificationService.sendNotification(user.id, 'Task Created')
    } catch {
      console.error('Just log the error and continue flow')
    }
  }
}
```

### Executing a test body while keeping time measurement as precise as possible

Adapted from https://github.com/isiahmeadows/thallium/blob/master/lib/core/tests.js#L369-L378

This is a case where things get a little awkward and boiilerplatey without it. In this case, I literally had to write a utility similar to what I'm proposing here just to better structure my code.

```js
class Context {
  // ...

  invokeInit(count) {
    const test = this.root.current

    test.locked = false
    const start = Date.now()
    const tryBody = try (0, test.callback)()
    const syncEnd = Date.now()

    // Note: synchronous failures are test failures, not fatal errors.
    if (tryBody.caught) {
      if (count < test.attempts) return this.invokeInit(count + 1)
      test.locked = true
      return Promise.resolve(new Result(syncEnd - start, true, tryBody.error))
    }

    const tryThen = try getThen(tryBody.value)

    if (tryThen.caught) {
      if (count < test.attempts) return this.invokeInit(count + 1)
      test.locked = true
      return Promise.resolve(new Result(syncEnd - start, true, tryThen.error))
    }

    if (typeof tryThen.value !== "function") {
      test.locked = true
      return Promise.resolve(new Result(syncEnd - start, false))
    }

    return new Promise(resolve => {
      let state = new AsyncState(this, start, resolve, count)
      const result = try invokeThen(tryThen.value, tryBody.value,
        () => {
          if (state == null) return
          state.finish(false)
          state = undefined
        },
        e => {
          if (state == null) return
          state.finish(true, e)
          state = undefined
        })

      if (state == null) return
      if (result.caught) {
          state.finish(true, result.error)
          state = undefined
          return
      }

      // Set the timeout *after* initialization. The timeout will likely be
      // specified during initialization.
      const maxTimeout = test.timeout || Constants.defaultTimeout

      // Setting a timeout is pointless if it's infinite.
      if (maxTimeout !== Infinity) {
        state.timer = setTimeout(() => {
          if (state == null) return
          state.finish(true, new Error(`Timeout of ${maxTimeout} reached`))
          state = undefined
        }, maxTimeout)
      }
    })
  }
}
```

Using `try`/`catch` instead:

```js
class Context {
  // ...

  invokeInit(count) {
    const test = this.root.current

    test.locked = false
    let caught = false
    let tryBody

    const start = Date.now()
    try {
      tryBody = (0, test.callback)()
    } catch (e) {
      // Note: synchronous failures are test failures, not fatal errors.
      const syncEnd = Date.now()
      if (count < test.attempts) return this.invokeInit(count + 1)
      test.locked = true
      return Promise.resolve(new Result(syncEnd - start, true, e))
    }
    const syncEnd = Date.now()
    let tryThen

    try {
      tryThen = getThen(tryBody.value)
    } catch (e) {
      if (count < test.attempts) return this.invokeInit(count + 1)
      test.locked = true
      return Promise.resolve(new Result(syncEnd - start, true, e))
    }

    if (typeof tryThen !== "function") {
      test.locked = true
      return Promise.resolve(new Result(syncEnd - start, false))
    }

    return new Promise(resolve => {
      let state = new AsyncState(this, start, resolve, count)
      try {
        try {
          invokeThen(tryThen.value, tryBody.value,
            () => {
              if (state == null) return
              state.finish(false)
              state = undefined
            },
            e => {
              if (state == null) return
              state.finish(true, e)
              state = undefined
            })
        } finally {
          if (state == null) return
        }
      } catch {
        state.finish(true, e)
        state = undefined
        return
      }

      // Set the timeout *after* initialization. The timeout will likely be
      // specified during initialization.
      const maxTimeout = test.timeout || Constants.defaultTimeout

      // Setting a timeout is pointless if it's infinite.
      if (maxTimeout !== Infinity) {
        state.timer = setTimeout(() => {
          if (state == null) return
          state.finish(true, new Error(`Timeout of ${maxTimeout} reached`))
          state = undefined
        }, maxTimeout)
      }
    })
  }
}
```

## Rationale

The [introduction](#introduction) covers it pretty well. But there's a few design decisions I want to go over.

### Why an object?

Two reasons:

1. It's easier to manage as a first-class citizen if you need to. Not all use cases boil down to in the small, and the names get invaluable if you use them anywhere else.
2. It's not ambiguous which field corresponds to what. Some might associate the first item as the value, some the second, and an object allows both ways while an array doesn't.

### Why not just an error + data pair?

Because it's lossy and `{error: undefined, value: undefined}` could be reasonably parsed as either an error or success depending on which value you check first. It's possible `undefined` could be thrown, and if you use `!= null` to track the presence/absence of an exception, it will lead you into problems. (In simpler cases, this won't come up, but it does come up frequently in advanced cases.)

Note that the differing objects do mean you *could* still do `const {caught, error, value} = try ...`. You can have informative names; you just have to check a third variable each time.

### Why an operator and not a function?

A function provides a *lot* more overhead, and while it's polyfillable, it's not ideal: you'd need separate variants for sync results and async values, and generators for each variant. Also, this naturally glides right in with the `try`/`catch` *statement*, which is itself already present as syntax and not as a special function of some sort.

## Language precedent

There's a few languages that themselves implement exception/error handling like this normally.

- Lua does almost exactly this via its built-in [`pcall`](https://www.lua.org/manual/5.3/manual.html#pdf-pcall). `pcall(func, ...)` calls `func(...)` and returns `true, result, ...` for success, `false, error` for caught errors. (They support multiple return values, but you can't throw multiple errors at once.) So this exists as pretty strong precedent.
- Perl does similar with `eval { ... }` + `$@`. `eval { ... }` returns `true` on success and `false` on error, and `$@` returns the caught error (if applicable). A common idiom `eval { ... } or do { ... }` provides a `try`/`catch`-like syntax similar to languages like C++ and Java, and [a pod exists to add a much more traditional `try`/`catch` statement common among most C-like languages supporting exceptions](https://metacpan.org/pod/Error).
- Rust does simple error handling via returning `Result<T, E>` (with `Ok(T)` and `Err(E)` variants), but its [`std::panic::catch_unwind`](http://doc.rust-lang.org/std/panic/fn.catch_unwind.html) catches panics (its stack-unwinding exceptions) and converts them to an `Ok(result)` on success, `Err(error)` on error. They recommend using the `Result<T, E>` directly for what you would normally use `try`/`catch` for, and only this if you have no other option (like avoiding UB with C interop).
- And of course, Go uses the `err, data` idiom where `err` is a possible error (or `nil` on success) and `data` is the result if successful (or `nil` on error). But this [runs into a pitfall I address in my rationale](#why-an-operator-and-not-a-function), and so I avoided that idiom specifically even though it was partial inspiration for this proposal.

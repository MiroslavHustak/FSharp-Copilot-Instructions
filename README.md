<!-- Information for humans only:

These coding instructions for universal-approximation-theorem-based copilots originate in my F# coding guidelines:
https://github.com/MiroslavHustak/FSharp-Coding-Guidelines
-->

# F# Copilot Instructions

---

## 0. How this document should be used

- This file **is the harness**, not a style suggestion to weigh against convenience. Treat every rule below as binding context for anything you generate, refactor, or review in this codebase.
- **Scope.** These rules target typical F# application code. They do not apply to special cases such as highly performant code, heavy data processing, game development, or graphics. If you believe you are in one of those cases, say so explicitly instead of silently deviating.
- **Be proactive, not just reactive.** If you notice code that violates these rules (a stray `for` loop, an untyped `ignore`, a `.Value` on an `Option`, a partial function such as `List.head`, an impure function not wrapped in `Impure`, an exception or null leaking past an interop boundary, a class that isn't at an interop edge), flag or fix it even if you weren't explicitly asked to, rather than silently extending the bad pattern because that's what the surrounding code does.
- **Surface conflicts instead of quietly bending the rules.** If a request can't be satisfied without breaking a philosophy rule (e.g. it genuinely needs a member outside an interop boundary), say so explicitly and propose the compliant alternative rather than complying silently or silently ignoring the rule.
- **Prefer current, well-vetted idioms over stale patterns.** When a newer feature of a preferred library (Thoth, FsHttp) or a newer `result{}`/`option{}` CE capability cleanly replaces an older workaround, suggest the update rather than defaulting to whatever pattern is most familiar.
- **Use the project's own helpers; do not reinvent them.** `Impure`/`runImpure` (§4) and `Option.ofNull'`/`Option.toResult` (§3) are defined in this codebase and reproduced in this file. Never substitute `IO`, `Option.ofObj`, `Option.ofNull`, `Option.ofNullable` or ad-hoc equivalents.
- **Tag significant copilot-generated sections** (e.g. with a comment) so that human reviewers can apply extra scrutiny. Generated code is a starting point, not a finished product.
- **Verify and be honest about it.** Follow §14 (self-review pass and strict claims policy) for every non-trivial piece of code you produce.
- Treat this as a living contract: if you can't comply with a rule for a concrete technical reason, state the reason inline (as required in §9 for non-tail-recursive functions) rather than leaving the deviation unexplained.

## 1. Philosophy — non-negotiable

- **Pure functional style.** No classes, inheritance, or interfaces, *except* where required for interop with .NET or external frameworks (e.g. Fabulous, TorchSharp, Elmish.WPF, Android).
- **No mixed paradigms.** Don't reach for a class or mutable field just because it's convenient. If it's not an interop boundary, it's wrong.
- **No mutable state**, except when unavoidable due to interop with .NET or external frameworks.
- When an interop boundary genuinely requires OOP (e.g. implementing an Android `Activity` or a .NET interface), keep the OOP surface as thin as possible and delegate immediately into functional code.
- **Functions, not members.** Keep records, DUs and SCDUs purely as data and operate on them with separate functions. Exception: members on SCDUs in custom CEs, where they are unavoidable by design.
- **Separate data from operations on data.**
- **Code organisation.** A single logical unit that gives the big picture of a component stays in one file and is never split. Code implementing one complete MVU (Model-View-Update) per UI component is one logical unit and must not be split. If a file seems too big, suggest separating independent logical units or nested MVU components instead of splitting a single one.
- **Single-direction dependency.** Avoid the `and` keyword and recursive namespaces. Exception: `and` between mutually recursive functions that form one inseparable algorithm (e.g. a CPS pair).

## 2. Control flow

- **`match` (and active patterns), never `if...then...else`**. This includes boolean branches; prefer matching on a tuple of booleans over nested `if`:
  ```fsharp
  match activity.IsFinishing, activity.IsDestroyed with
  | true, _ | _, true -> ...
  | false, false -> ...
  ```
- **No `for`/`while` loops.** Use `List.map`/`iter`/`fold`, `Seq` (with care around laziness), or tail-recursive/CPS functions (with an accumulator where needed).
- No LINQ, no query expressions.

## 3. Error handling

- **Errors are values, not exceptions.** Propagate with `Result`/`Option`, composed via `result{}`, `option{}`, `asyncResult{}`, `asyncOption{}` (and custom monadic CEs where appropriate). Do not raise exceptions.
- **No partial functions.** A partial function is an exception in disguise: it throws on input it cannot handle (empty collection, missing key, out-of-range index). Do not use it in committed code. Use the total variant returning `Option`/`Result`, or pattern match explicitly.

  | Do not use | Use instead |
  |---|---|
  | `List.head`, `Seq.head`, `Array.head` | `List.tryHead`, `Seq.tryHead`, `Array.tryHead` |
  | `List.last`, `Seq.last`, `Array.last` | `List.tryLast`, `Seq.tryLast`, `Array.tryLast` |
  | `List.item`, `Seq.item`, `Array.item`, `xs.[i]` | `List.tryItem`, `Seq.tryItem`, `Array.tryItem` |
  | `List.find`, `Seq.find`, `Array.find` | `List.tryFind`, `Seq.tryFind`, `Array.tryFind` |
  | `List.findIndex` | `List.tryFindIndex` |
  | `List.pick`, `Seq.pick` | `List.tryPick`, `Seq.tryPick` |
  | `Map.find`, `dict.[key]` | `Map.tryFind` (convert a .NET `Dictionary` lookup at the boundary with `TryGetValue`) |
  | `Seq.exactlyOne`, `List.exactlyOne` | pattern match on the list (`[ x ]`, `[]`, `_ :: _`) |
  | `List.max`/`min`/`reduce`/`average` on a possibly empty collection | pattern match on emptiness first |
  | `failwith`, `failwithf`, `invalidArg`, `raise`, `Option.get`, `.Value` | return `Error`/`None` |
  | `Int32.Parse` and similar throwing parsers | `TryParse`, wrapped at the boundary and converted to `Option`/`Result` |

  Also guard against division by zero and other operations that throw on valid-looking input. If you believe a partial function is genuinely safe in a particular place, do not use it silently: either restructure so it is not needed, or add a comment stating why it cannot throw (compare §9).
- **Keep the combinator vocabulary small.** When composing without a CE, prefer: `Option.map/bind/iter`, `Result.map/bind`, `Option.defaultValue`, `Option.orElseWith`, `Option.toResult`, `Result.defaultWith`, `Result.defaultValue`, `Result.sequence`, `Result.either`, `Result.mapError`.
- **Custom CEs:** `Bind` must satisfy the monad laws. If it deliberately breaks them, use an alias to avoid confusion.
- **Exceptions are converted at the boundary.** Any .NET/Android API that can throw or return null gets wrapped immediately. The exception or null must not leak past the first function that calls into the framework.
  ```fsharp
  try
      activity.StartActivityForResult(intent, REQUEST_CODE)
  with
  | ex ->
      tcs.TrySetException(ex) |> ignore<bool>
  ```
  `try/with` at an interop boundary is correct and expected. It is not a violation of "no exception-based flow." The violation would be letting that exception propagate as control flow *into* pure application code instead of converting it to a `Result`/`Option`/`TrySetException` at the seam.
- **Nulls from .NET/Android are converted to `Option` at the boundary with `Option.ofNull'`**, used consistently for all nullable .NET types (reference types and `Nullable<T>`) instead of `Option.ofObj`, `Option.ofNull`, `Option.ofNullable`. If a `Nullable<T>` is passed, the wrapper is preserved inside `Some` and causes a type mismatch downstream. This is intentional: it signals that `Option.ofNullable` is needed there.

  `Option.ofNull'` is **not** part of FSharp.Core or FsToolkit. It is this project's own helper, defined as follows (do not redefine it, do not replace it with a different null-check):

  ```fsharp
  namespace Extensions

  open FsToolkit.ErrorHandling

  [<RequireQualifiedAccess>]
  module Option =

      let inline internal ofNull' (value : 'nullableValue) =
          match System.Object.ReferenceEquals(box value, null) with //boxing a null Nullable<'T> produces an actual null reference
          | true  -> None
          | false -> Some value

      let internal toResult err =
          function
          | Some value -> Ok value
          | None       -> Error err
  ```

  Usage:

  ```fsharp
  let private disposeIntent (data : Intent) : unit =
      match Option.ofNull' data with
      | None   -> ()
      | Some d -> try d.Dispose() with | _ -> ()
  ```
- **Never introduce nulls into F# code.** Do not assign `null` to strings, arrays, .NET collections or classes, and do not use `[<AllowNullLiteral>]` on your own types. Represent absence with `Option` (or an empty collection/string where appropriate, with caution).
- **No `.Value` and no `Option.get` on `Option`** in committed code. Use pattern matching, `Option.map/bind/iter`, `Option.defaultValue`, `Option.orElseWith`, or `option{}`.
- **No `Nullable<T>` in F# code.** Convert to `Option<T>` at the interop boundary. Hand-rolled `Some v -> Nullable(v) | None -> Nullable()` conversions are prohibited.
- **Reflection-free code.** No `System.Reflection` APIs (`GetMethod`, `GetProperty`, `Invoke`, `Activator.CreateInstance`, …), no `FSharp.Reflection`, no reflection-driven features such as `%A`. Type tests (`:?`), `nameof`, and compiler-consumed attributes are not reflection. Reflection hidden behind a clean library API is acceptable only if it does not leak into our code, does not hurt critical paths, is trimming/AOT-compatible where relevant, no reflection-free alternative of comparable quality exists, and values it produces are validated at the boundary.
- **`%A` is prohibited in production code.** It is allowed only in test code, only for diagnostics, and sparingly. Implicit `%A` (`string`, `%O`, or interpolation on records, unions, exceptions) counts too: format fields explicitly or override `ToString`.
- **Explicit deserialization over implicit mapping.** With Thoth.Json use explicit decoders/encoders (`Decode.object`/`Decode.field` style), never the `Auto` ones. Type providers are acceptable where the data shape is under our control or a representative sample is guaranteed. For external or untrusted payloads use explicit decoders.
- **AOT/trimming** (only for projects explicitly targeting Native AOT or trimming): additionally no assembly scanning/auto-discovery, no runtime code generation (`Reflection.Emit`, `DynamicMethod`, dynamic proxies), no runtime type discovery/loading (`Assembly.Load`, `Type.GetType(string)`, `MakeGenericType`), and Thoth.Json explicit codecs only.

## 4. Impurity — `Impure` is mandatory

Separate pure and impure functions where possible, and do not hide impurity in function signatures. The concern is impurity that can *leak invisibly* across a boundary someone else relies on, so the wrapper applies where that risk actually exists, not everywhere.

- **Public and internal functions**: impurity must be visible in the signature. **Wrap the function's return type with `Impure`** (defined below); a caller elsewhere in the module/assembly has no other way to know the function is impure, so the type itself must carry that information. This is mandatory.
- **Private functions**: same rule, same wrapper (here, a human developer may discontinue the `Impure` wrapper later, but always leave the decision up to them).
- **Local (embedded/nested) functions**: do **not** wrap with `Impure`. A function defined inside the body of the function that calls it has no hidden surface: its impurity is directly visible in the enclosing code, right where it's used. Wrapping it would just add ceremony without preventing anything, since there's nothing for it to hide from.
- Execute an `Impure` value only with `runImpure` (or `runImpureAsync` inside `async {}`), at the outermost point that genuinely needs the result. Do not invent other wrapper types (`IO`, `Task`-as-effect, etc.) and do not redefine `Impure`.

The project's own definitions (namespace `Core`); use them as they are:

```fsharp
namespace Core

module Haskell_Monad_Simulation =

    type [<Struct>] internal Impure<'a> = Impure of (unit -> 'a) // wrapping custom type simulating Haskell's monads (such as IO Monad, Writer etc) - without the monad, of course

    let internal runImpure (Impure action) = action ()
    let internal runImpureAsync (Impure action) : Async<'a> = async { return action () }
```

Usage:

```fsharp
let internal readConfig (path : string) : Impure<Result<string, ParsingAndDownloadingErrors>> =

    Impure (fun () ->

        try
            System.IO.File.ReadAllText path |> Ok
        with
        | _ -> Error FileDownloadError
    )

// at the point where the effect is actually needed
let result = runImpure <| readConfig path
```

The remaining members of that module (`bind`, `impureFn`, `RealWorld`, `IO2`, `runImpure2`, `bind2`, `returnIO`, `bindIO`, `mapIO`) and the `FreeMonad` module are educational / not-yet-used code. Do not use them in new code and do not copy them as patterns.

## 5. Preferred libraries over .NET ones

- Thoth (explicit decoders/encoders only)
- FsHttp
- FsToolkit.ErrorHandling (for `result{}`, `option{}`, `asyncResult{}`, `asyncOption{}`)
- Type providers for CSV, XML and JSON are preferred over equivalent .NET libraries, subject to the conditions in §3 (Explicit deserialization over implicit mapping).

## 6. Type safety

- Prefer single-case DUs (SCDUs) over bare primitives where it clarifies intent (type-driven development), and units of measure (or FSharp.UMX phantom types) for physical quantities or otherwise-identical primitive IDs.
- Use `ignore<'T>` (not bare `ignore`) after side-effecting calls whose result is intentionally discarded. This catches partial-application bugs at compile time.

## 7. Async

- `async {}` workflows. C#-style `async`/`await` is prohibited, except where the .NET API surface (e.g. `Task<'T>` from an Android callback) makes `Task` unavoidable; convert to `Async` with `Async.AwaitTask` at the boundary.
- If an asynchronous variant of an API exists, prefer it (cancellation and non-blocking constructs then need minimal refactoring), with caution when used inside parallel loops.
- `Async.Parallel` for concurrency; `Task`/`Array.Parallel` only for CPU-bound, performance-sensitive work or at .NET boundaries.

## 8. Collections & strings

- Immutable collections (`List`, `Seq`, `Set`, `Map`) by default; exercise caution with lazy `Seq`. `Array` only for performance-sensitive or interop reasons, converted to an immutable collection at the functional boundary.
- Type-safe `sprintf` (never `%A` in production, see §3) for string composition unless there's a compelling reason to use `Path.Combine`, `String.concat`, etc.

## 9. Recursive functions

- Use recursion only when there is no viable alternative such as functions on collections (`List.map`/`fold`/etc.).
- Ensure all recursive functions avoid stack overflow. Prefer tail-recursive functions (using an accumulator where needed) or CPS-compliant functions written so a trampoline can be applied.
- **Caution:** .NET's tail-call optimization is not guaranteed on every runtime/architecture (notably ARM and some AOT/mobile targets). Where tail-call elimination cannot be relied on, prefer an explicit accumulator-based tail-recursive loop over bare CPS, or bound recursion depth via `Seq` (lazy evaluation) instead of direct recursion.
- In the rare case a recursive function is neither tail-recursive nor CPS-compliant, always add a comment explaining why it's safe (e.g. provably bounded/shallow input, depth capped elsewhere). Don't leave the exception unjustified.

## 10. Formatting style

Keep this formatting style:

```fsharp
let internal execute dispatch (token : CancellationToken) =

    Impure (fun () ->

        async
            {
                try
                    use cts = CancellationTokenSource.CreateLinkedTokenSource token

                    let token2 = cts.Token

                    let reportProgress (progressValue : float, totalProgress : float) =
                        match token2.IsCancellationRequested with
                        | true  -> dispatch <| Progress (0.0, 1.0)
                        | false -> dispatch <| Progress (progressValue, totalProgress)

                    match token2.IsCancellationRequested with
                    | true
                        ->
                        return dispatch NavigateHome

                    | false
                        ->
                        match Helpers.ConnectivityWithDebouncing.isNowConnected () with
                        | false
                            ->
                            return dispatch NoInternet

                        | true
                            ->
                            do! Async.SwitchToThreadPool()
                            umMiliSecondsToInt32 >> cts.CancelAfter <| timeoutMs

                            let! result =
                                async
                                    {
                                        return
                                            stateReducer
                                                <| token2
                                                <| kodisPathTemp4 ()
                                                <| fun msg -> IterationMsg >> dispatch <| msg
                                                <| fun b   -> CancelButtonVisible >> dispatch <| b
                                                <| reportProgress
                                            |> runImpure
                                    }

                            match token2.IsCancellationRequested with
                            | true  -> return dispatch NavigateHome
                            | false -> return Completed >> dispatch <| result

                with
                | ex
                    ->
                    use cts = CancellationTokenSource.CreateLinkedTokenSource token

                    match runImpure <| isCancellationGeneric LetItBe StopDownloading TimeoutError FileDownloadError cts.Token ex with
                    | err
                        when err = StopDownloading
                        ->
                        //runImpure (postToLog2 <| string ex.Message <| " StopDownloading #9999 Kodis Canopy")
                        match Helpers.ConnectivityWithDebouncing.isNowConnected () with
                        | false -> return dispatch NoInternet
                        | true  -> return dispatch NavigateHome
                    | _
                        ->
                        //runImpure (postToLog2 <| string ex.Message <| " #XElmish_Kodis4_Critical_Error")
                        return dispatch NoInternet
            }
    )
```

## 11. Data handling

- **No ORMs or micro-ORMs** (e.g. Entity Framework Core). Use plain SQL or SQL type providers.
- All plain SQL must be strictly **parameterised**. SQL injection is unacceptable.
- Exercise caution with SQL type providers (compilation times, schema-mapping limits on large databases); vanilla SQL is the alternative.

## 12. Testing

- Pure functions are assumed correct by design, especially with type-driven development. Unit tests for them are optional; use integration tests where necessary and **property-based testing (PBT), which is mandated**.
- **No tautological tests.** A test must express the expected behavior (and, in a short comment or test name, why it matters), not re-implement the function under test. In particular, a property that merely recomputes the implementation's result, or an assertion that cannot fail whatever the implementation does, is worthless. Prefer properties that state an independent invariant (round-trip, idempotence, ordering, conservation of a quantity, agreement with a simpler reference model).
- A test only counts as evidence of correct behavior if it would fail when that behavior is broken. If you cannot show or reason concretely about how a test would fail, do not claim it verifies the behavior (see §14).
- `%A` is allowed in test code for diagnostics only, sparingly, and not in loops, generators, or on large collections.

## 13. Advanced FP concepts & FSharpPlus

- Prefer simpler, idiomatic functional constructs. Advanced concepts (isomorphism, cardinality, state monads, free monads) and the FSharpPlus library are discouraged in production code for readability and maintainability.
- Exceptions: free monads are acceptable in console applications; all of these may be used freely in prototypes, proofs of concept, experiments, and other disposable or one-off code.

## 14. Verification and honesty

Generated code that looks fine can still hide serious problems, and the human reviewer should not have to find them alone.

- **Self-review pass before presenting code.** After writing any non-trivial code (more than a few lines), re-read it once against this file before showing it. Check at least for:
  - `if...then...else`, `for`/`while`, LINQ or query expressions
  - `.Value`, `Option.get`, other partial functions (§3), `failwith`/`raise`
  - nulls, `[<AllowNullLiteral>]`, `Nullable<T>`, and nulls or exceptions leaking past a boundary
  - an impure public/internal/private function without `Impure`, or a use of `IO`/`runIO`
  - bare `ignore`, `%A` (including implicit via `string`/`%O`/interpolation on records, unions, exceptions)
  - classes, members, mutability outside an interop boundary
  - recursion that is neither tail-recursive nor CPS-compliant and has no justifying comment
  - formatting deviating from §10

  Fix what you find. If a deviation remains for a concrete technical reason, report it explicitly next to the code rather than leaving it for the human to discover.
- **Strict claims policy.** Do not state or imply that code "works", "compiles", "is tested", "is correct" or "is safe" unless you actually compiled it and ran the tests in this session and saw the result. If you did not or could not, say so plainly.
- **List what was not verified.** End any non-trivial response with a short note of what you did *not* verify (e.g. not compiled, tests not run, edge cases not exercised, behavior on empty input not checked, Android/runtime behavior not checked).
- **No claims from tautological tests.** A test you wrote may be cited as evidence only if it satisfies §12.
- When uncertain, say you are uncertain. Do not fill gaps with confident-sounding guesses about APIs, library behavior, or framework semantics.

---

## Quick reference (apply to all responses)

- Functional-only: no classes/interfaces/inheritance, no mutable state, except at .NET/Android/framework interop boundaries; keep that surface thin, delegate into functional code immediately. Functions, not members; data separate from operations.
- Control flow: `match`/active patterns, never `if...then...else` (incl. boolean branches; match tuples of bools). No `for`/`while`, no LINQ/query exprs; use map/iter/fold or tail-recursion/CPS with accumulator.
- Errors are values: use `Result`/`Option` (`result{}`, `option{}`, `asyncResult{}`, `asyncOption{}`), no raised exceptions. Convert exceptions/nulls to `Result`/`Option` at any .NET/Android boundary (try/with there is correct) using the project's `Option.ofNull'`; never let them leak into pure code.
- No partial functions (`List.head`, `List.item`, `List.find`, `Map.find`, `Seq.exactlyOne`, indexers `xs.[i]`, `failwith`, `Option.get`, throwing parsers): use `tryHead`/`tryItem`/`tryFind`/`tryFindIndex`/`tryPick`/`Map.tryFind`/`TryParse` or pattern matching (table in §3).
- Never introduce nulls, `[<AllowNullLiteral>]`, `Nullable<T>`, `.Value` or `Option.get` on `Option`.
- No reflection; no `%A` in production (incl. implicit via `string`/`%O`/interpolation on records, unions, exceptions). Thoth explicit decoders only, never `Auto`.
- **`Impure` is mandatory** for impure public/internal/private functions: return type `Impure<'a>` (`type [<Struct>] internal Impure<'a> = Impure of (unit -> 'a)`), executed via `runImpure` / `runImpureAsync`. Local nested functions are not wrapped. Never use `IO`/`runIO`. Keep pure/impure separated.
- Use the project's `Option.ofNull'` and `Option.toResult` (definitions in §3); never `Option.ofObj`/`ofNull`/`ofNullable`.
- Prefer SCDUs over bare primitives; units of measure/FSharp.UMX for distinct numeric IDs.
- Use `ignore<'T>` (typed), never bare `ignore`, after discarded side-effecting calls.
- `async{}`, never C#-style async/await; convert `Task` via `Async.AwaitTask` at boundaries; prefer async API variants. `Async.Parallel` for concurrency.
- Immutable collections (List/Seq/Set/Map) by default; `Array` only when justified.
- Type-safe `sprintf` for strings, not concatenation methods, unless justified.
- Recursion only when collection functions can't do the job; must be tail-recursive/CPS-compliant, or explicitly justified as safely bounded. Avoid `and` except for one inseparable mutually recursive algorithm.
- Don't split a single logical unit (incl. one complete MVU per UI component) across files.
- No ORMs; parameterised plain SQL or SQL type providers. PBT for pure functions, no tautological tests. No FSharpPlus/advanced FP in production.
- Before presenting non-trivial code: do a self-review pass against these rules (§14) and report remaining deviations explicitly.
- Never claim code works, compiles, is tested or is safe unless you actually compiled/ran it; end non-trivial responses with what was not verified.
- Formatting: one match case per line, `->` often on its own indented line; don't compact to inline match expressions.
- Tag significant copilot-generated sections for reviewers.

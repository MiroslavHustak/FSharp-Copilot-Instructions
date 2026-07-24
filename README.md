Information for humans only: 

These coding instructions for universal-approximation-theorem-based copilots originate in my F# coding guidlines:
https://github.com/MiroslavHustak/FSharp-Coding-Guidelines



# F# Copilot Instructions

--

## 0. How this document should be used

- This file **is the harness**, not a style suggestion to weigh against convenience. Treat every rule below as binding context for anything you generate, refactor, or review in this codebase.
- **Be proactive, not just reactive.** If you notice code that violates these rules — a stray `for` loop, an untyped `ignore`, an exception leaking past an interop boundary, a class that isn't at an interop edge — flag or fix it even if you weren't explicitly asked to, rather than silently extending the bad pattern because that's what the surrounding code does.
- **Surface conflicts instead of quietly bending the rules.** If a request can't be satisfied without breaking a philosophy rule (e.g. it genuinely needs a member outside an interop boundary), say so explicitly and propose the compliant alternative rather than complying silently or silently ignoring the rule.
- **Prefer current, well-vetted idioms over stale patterns.** When a newer feature of a preferred library (Thoth, FsHttp) or a newer `result{}`/`option{}` CE capability cleanly replaces an older workaround, suggest the update rather than defaulting to whatever pattern is most familiar.
- Treat this as a living contract: if you can't comply with a rule for a concrete technical reason, state the reason inline (as required in §9 for non-tail-recursive functions) rather than leaving the deviation unexplained.

## 1. Philosophy — non-negotiable

- **Pure functional style.** No classes, inheritance, or interfaces, *except*
  where required for interop.
- **No mixed paradigms.** Don't reach for a class or mutable field just
  because it's convenient — if it's not an interop boundary, it's wrong.
- When an interop boundary genuinely requires OOP (e.g. implementing an
  Android `Activity` or a .NET interface), keep the OOP surface as thin as
  possible and delegate immediately into functional code.

## 2. Control flow

- **`match`, never `if...then...else`**. This includes boolean branches — prefer matching on a
  tuple of booleans over nested `if`:
  ```fsharp
  match activity.IsFinishing, activity.IsDestroyed with
  | true, _ | _, true -> ...
  | false, false -> ...
  ```
- **No `for`/`while` loops.** Use `List.map`/`iter`/`fold`, `Seq` (with
  care around laziness), or tail-recursive/CPS functions.
- No LINQ, no query expressions.

## 3. Error handling

- **Errors are values, not exceptions.** Propagate with `Result`/`Option`,
  composed via `result{}`, `option{}`, `asyncResult{}`, `asyncOption{}`.
- **Exceptions are converted at the boundary.** Any .NET/Android API that can
  throw or return null gets wrapped immediately — the exception or null must
  not leak past the first function that calls into the framework.
  ```fsharp
  try
      activity.StartActivityForResult(intent, REQUEST_CODE)
  with
  | ex ->
      tcs.TrySetException(ex) |> ignore<bool>
  ```
  `try/with` at an interop boundary is correct and expected — it is not a
  violation of "no exception-based flow." The violation would be letting
  that exception propagate as control flow *into* pure application code
  instead of converting it to a `Result`/`Option`/`TrySetException` at the
  seam.
- **Nulls from .NET/Android are handled with `Option`.** Use direct null checks only where there's no other way
  (typically deserializing a nullable reference type at an interop edge):
  ```fsharp
  let private disposeIntent (data : Intent) : unit =
      match data with
      | null -> ()
      | d    -> try d.Dispose() with | _ -> ()
  ```

## 4. Impurity

Do not hide impurity in function signatures. The concern is impurity that can *leak invisibly* across a boundary someone else relies on — so the wrapper applies where that risk actually exists, not everywhere.

- **Public and internal functions** — impurity must be visible in the signature. Wrap the function's return type with the `IO` type below; a caller elsewhere in the module/assembly has no other way to know the function is impure, so the type itself must carry that information.
- **Private functions** — same rule, same wrapper (here, a human developer may discontinue the IO wrapper later, but always leave the decision up to them).
- **Local (embedded/nested) functions** — do **not** wrap with `IO`. A function defined inside the body of the function that calls it has no hidden surface: its impurity is directly visible in the enclosing code, right where it's used. Wrapping it would just add ceremony without preventing anything, since there's nothing for it to hide from.

```fsharp
module Haskell_IO_Monad_Simulation =

    type [<Struct>] internal IO<'a> = IO of (unit -> 'a) // wrapping custom type simulating Haskell's IO Monad (without the monad, of course)

    let internal runIO (IO action) = action ()
```

## 5. Preferred libraries over .NET ones

- Thoth
- FsHttp

## 6. Type safety

- Prefer single-case DUs (SCDUs) over bare primitives where it clarifies
  intent, and units of measure (or FSharp.UMX phantom types) for physical
  quantities or otherwise-identical primitive IDs.
- Use `ignore<'T>` (not bare `ignore`) after side-effecting calls whose
  result is intentionally discarded — this catches partial-application bugs
  at compile time.

## 7. Async

- `async {}` workflows, not C#-style `async`/`await`, except where the .NET
  API surface (e.g. `Task<'T>` from an Android callback) makes `Task`
  unavoidable — convert to `Async` with `Async.AwaitTask` at the boundary.
- `Async.Parallel` for concurrency; `Task`/`Array.Parallel` only for
  CPU-bound, performance-sensitive work.

## 8. Collections & strings

- Immutable collections (`List`, `Seq`, `Set`, `Map`) by default; `Array`
  only for performance-sensitive or interop reasons, converted to an
  immutable collection at the functional boundary.
- Type-safe `sprintf` for string composition unless there's a specific
  reason to use `Path.Combine`, `String.concat`, etc.

## 9. Recursive functions

- Use recursion only when there is no viable alternative such as functions
  on collections (`List.map`/`fold`/etc.).
- Ensure all recursive functions avoid stack overflow. Prefer
  tail-recursive functions (using an accumulator where needed) or
  CPS-compliant functions written so a trampoline can be applied.
- **Caution:** .NET's tail-call optimization is not guaranteed on every
  runtime/architecture (notably ARM and some AOT/mobile targets). Where
  tail-call elimination cannot be relied on, prefer an explicit
  accumulator-based tail-recursive loop over bare CPS, or bound recursion
  depth via `Seq` (lazy evaluation) instead of direct recursion.
- In the rare case a recursive function is neither tail-recursive nor
  CPS-compliant, always add a comment explaining why it's safe (e.g.
  provably bounded/shallow input, depth capped elsewhere) — don't leave
  the exception unjustified.

## 10. Formatting style

Keep this formatting style:

```fsharp
let internal execute dispatch (token : CancellationToken) =

    IO (fun () ->

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
                                            |> runIO
                                    }

                            match token2.IsCancellationRequested with
                            | true  -> return dispatch NavigateHome
                            | false -> return Completed >> dispatch <| result

                with
                | ex
                    ->
                    use cts = CancellationTokenSource.CreateLinkedTokenSource token

                    match runIO <| isCancellationGeneric LetItBe StopDownloading TimeoutError FileDownloadError cts.Token ex with
                    | err
                        when err = StopDownloading
                        ->
                        //runIO (postToLog2 <| string ex.Message <| " StopDownloading #9999 Kodis Canopy")
                        match Helpers.ConnectivityWithDebouncing.isNowConnected () with
                        | false -> return dispatch NoInternet
                        | true  -> return dispatch NavigateHome
                    | _
                        ->
                        //runIO (postToLog2 <| string ex.Message <| " #XElmish_Kodis4_Critical_Error")
                        return dispatch NoInternet
            }
    )
```

---

## Quick reference (apply to all responses)

- Functional-only: no classes/interfaces/inheritance except at .NET/Android interop boundaries; keep that surface thin, delegate into functional code immediately.
- Control flow: `match`, never `if...then...else` (incl. boolean branches — match tuples of bools). No `for`/`while`, no LINQ/query exprs; use map/iter/fold or tail-recursion.
- Errors are values: use `Result`/`Option` (`result{}`, `option{}`, `asyncResult{}`). Convert exceptions/nulls to `Result`/`Option` at any .NET/Android boundary (try/with there is correct) — never let them leak into pure code.
- Wrap impure functions: `type IO<'a> = IO of (unit -> 'a)`, run via `runIO`. Keep pure/impure separated.
- Prefer SCDUs over bare primitives; units of measure/FSharp.UMX for distinct numeric IDs.
- Use `ignore<'T>` (typed), never bare `ignore`, after discarded side-effecting calls.
- `async{}`, not C#-style async/await; convert `Task` via `Async.AwaitTask` at boundaries. `Async.Parallel` for concurrency.
- Immutable collections (List/Seq/Set/Map) by default; `Array` only when justified.
- Type-safe `sprintf` for strings, not concatenation methods, unless justified.
- Recursion only when collection functions can't do the job; must be tail-recursive/CPS-compliant, or explicitly justified as safely bounded.
- Formatting: one match case per line, `->` often on its own indented line; don't compact to inline match expressions.

---
layout: post
title:  "Aether - Total &amp; Partial Lenses for F#"
categories: lenses aether fsharp
---

I've just released a library ([SemVer.Net][semver-nuget]) for doing [SemVer 2.0.0][semver] compliant semantic versioning in .NET. At the moment it provides a simple SemanticVersion type and nothing else, but that type fully implements the standard and sorts all semantic versions correctly.

```fsharp
open System
open System.Collections.Generic
open System.Threading.Tasks
 
// Low Level
 
// Aliases
 
type OwinEnvironment = IDictionary<string, obj>
type OwinFunction = OwinEnvironment -> Async<OwinEnvironment>
type OwinApplicationDelegate = Func<OwinEnvironment, Task>
 
// Operators
 
let (>=>) (f1: OwinFunction) (f2: OwinFunction) : OwinFunction =
    fun e ->
        async {
            let! e' = f1 e
            return! f2 e' }
 
// Helpers
 
let toOwinApplicationDelegate (f: OwinFunction) : OwinApplicationDelegate =
    Func<_,_> (fun e -> Async.StartAsTask (f e) :> Task)
 
// High Level
 
// Aliases
 
type OwinMonad<'T> = OwinEnvironment -> Async<'T * OwinEnvironment>
 
// Monads
 
type OwinMonadBuilder () =
 
    member x.Return t : OwinMonad<_> = 
        fun s -> async.Return (t, s)
 
    member x.ReturnFrom f : OwinMonad<_> = 
        f
 
    member x.Zero () : OwinMonad<unit> = x.Return ()
 
    member x.Bind (m: OwinMonad<_>, k: _ -> OwinMonad<_>) : OwinMonad<_> =
        fun state -> 
            async {
                let! result, state = m state
                return! (k result) state }
 
    member x.Delay (f: unit -> OwinMonad<_>) : OwinMonad<_> = 
        x.Bind (x.Return (), f)
 
    member x.Combine (r1: OwinMonad<_>, r2: OwinMonad<_>) : OwinMonad<_> = 
        x.Bind (r1, fun () -> r2)
 
    member x.TryWith (body: OwinMonad<_>, handler: exn -> OwinMonad<_>) : OwinMonad<_> =
        fun state -> 
            try body state 
            with ex -> 
                handler ex state
 
    member x.TryFinally (body: OwinMonad<_>, handler) : OwinMonad<_> =
        fun state -> 
            try body state 
            finally handler ()
 
    member x.Using (resource: #IDisposable, body: _ -> OwinMonad<_>) : OwinMonad<_> =
        x.TryFinally (body resource, fun () ->
            match box resource with
            | null -> ()
            | _ -> resource.Dispose ())
 
    member x.While (guard, body: OwinMonad<_>) : OwinMonad<unit> =
        match guard () with
        | true -> x.Bind (body, fun () -> x.While (guard, body))
        | _ -> x.Zero ()
 
    member x.For (sequence: seq<_>, body: _ -> OwinMonad<_>) : OwinMonad<unit> =
        x.Using (sequence.GetEnumerator (),
            (fun enum ->
                x.While (enum.MoveNext, x.Delay (fun () ->
                    body enum.Current))))
 
let owin = OwinMonadBuilder ()
```

And some more content after...

[fparsec]: http://www.quanttec.com/fparsec/
[semver]: http://semver.org/spec/v2.0.0.html
[semver-nuget]: https://www.nuget.org/packages/SemVer.Net
[semver-github]: https://github.com/xyncro/semver.net


---
layout: post
title:  "SemVer.Net &mdash; Semantic Versioning for .NET"
categories: dotnet semver
---

I've just released a library ([SemVer.Net][semver-nuget]) for doing [SemVer 2.0.0][semver] compliant semantic versioning in .NET. At the moment it provides a simple SemanticVersion type and nothing else, but that type fully implements the standard and sorts all semantic versions correctly.

It's written in F# using [FParsec][fparsec] to write a full parser for the version format rather than a validating regex, to make sure the edge cases of the specification can be implemented properly. The code has been released as part of my organizational GitHub account (Xyncro) [here][semver-github] under an __MIT license__.

[fparsec]: http://www.quanttec.com/fparsec/
[semver]: http://semver.org/spec/v2.0.0.html
[semver-nuget]: https://www.nuget.org/packages/SemVer.Net
[semver-github]: https://github.com/xyncro/semver.net


---
layout: post
title:  "Thoughts on Hypermedia Standards &mdash; Part 1"
categories: hypermedia
---

Recently I've started to think about the next layer of abstraction for [Freya][freya], and what that might mean. At this point Freya allows you to work with HTTP from "raw" (ish) [OWIN][owin] data up to [WebMachine][webmachine]-style decision graphs (although in a more powerful and extensible way than WebMachine). The next level of abstraction up for Freya is (potentially) Hypermedia.

I'm not going to take up space here with an introduction to what Hypermedia is, or why you'd want it (there are [better sources][fielding-1]) so I'm going to assume a reasonable level of conceptual familiarity on the part of the reader (sorry!).

## The State of Play

Although in the long term I would like Freya to have solid support for multiple standards, realistically I have to pick one to implement *first*. Without further ado, the candidates at this point look like this (alphabetically, without prejudice):

* [Collection+JSON][collection+json]
* [HAL+JSON][hal+json]
* [JSON API][json api]
* [JSON-LD][json-ld] (optionally with [Hydra][hydra])
* [JSON-Schema][json-schema] with [JSON-Hyperschema][json-hyperschema]
* [Siren][siren]
* [UBER][uber]

The first thing that becomes obvious when researching these standards is that they *are not standards*. They are all at best in recommendation stage, and are liable to change and evolve regularly and rapidly. While this is never going to form all of a basis for adoption, it is useful to be able to compare the states that each proposed standard has reached, maturity, communities/organizations backing, etc.

## Progress

With the non-final/non-official state of each proposal in mind, here's an attempted summary of the progress/status of each:

| Proposal | Significant/Current Status | Organization | Activity |
|----------|----------------------------|--------------|----------|
| Collection+JSON | Approved IANA Registration | Individual/Ad-Hoc Community | Low |

[freya]: https://github.com/freya-fs/freya
[owin]: http://owin.org/
[webmachine]: https://github.com/basho/webmachine/wiki
[fielding-1]: http://www.ics.uci.edu/~fielding/pubs/dissertation/web_arch_domain.htm#sec_4_1_3

[collection+json]: http://amundsen.com/media-types/collection/
[hal+json]: http://stateless.co/hal_specification.html
[json api]: http://jsonapi.org/
[json-ld]: http://json-ld.org/
[hydra]: http://www.markus-lanthaler.com/hydra/
[json-schema]: http://json-schema.org/
[json-hyperschema]: http://json-schema.org/
[siren]: https://github.com/kevinswiber/siren
[uber]: https://rawgit.com/uber-hypermedia/specification/master/uber-hypermedia.html

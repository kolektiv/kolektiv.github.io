---
title: Using AMQP to build CQRS systems
tags:
- cqrs
- amqp
- development
---
Recently I've been working on building out a system which uses CQRS as a fairly major part of the architecture. I'm not going to go in to what CQRS is and isn't, or why you might want to use it. There's enough information out there now that it shouldn't be a black hole if you google it - Greg Young, Udi Dahan, Rinat Abdullin have all written sizable chunks and made them web available, along with many others.

We're usually interested in CQRS for one of a couple of reasons. Usually it has something to do with scaling or performance, although you might be interested in it more as a system which allows us to better practice DDD. If you care about scaling though, how are you going to go about scaling this?

This weekend I played around with using AMQP in a CQRS context. AMQP is interesting because it's not just striaghtforward messaging. The standard defines some behavioural aspects, and it defines the capabilities of an AMQP system above and beyond simply "send a receive a message" (so, more than just non-transactional point-to-point).

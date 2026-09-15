---
lang: en
---

I have been writing technical books for a long time. It started out as an itch that I needed to scratch and turned into a full-on rash. The cycle that drives me is _learn -> build -> teach_, which covers book writing. To be able to teach something to someone in simple terms, I need to gain deep understanding of it.

The following are the most recent and relevant books that pertain to my passions and day job of event-sourcing, distributed systems, secure untrusted computing, massive scale, and high througput.

## Real-World Event Sourcing

<img class="book-cover" src="/images/covers/real-world-event-sourcing.jpg" alt="Cover of Real-World Event Sourcing" width="500" height="600">

*Distribute, Evolve, and Scale Your Elixir Applications*. The Pragmatic Bookshelf, 2025.

Applications use streams of incoming data to create their own realities. This book is about learning to treat data as events and then interpreting those events to solve genuinely hard problems, from the first line of code through to production. It starts with the fundamental building blocks of an event-sourced system: commands, aggregates, projectors, process managers, injectors, and notifiers. From there it moves into the parts that only show up once you have real systems in the field, such as distributing an application across nodes, modeling failure, replaying history, evolving event schemas without breaking what already exists, securing and testing event-sourced systems, and scaling up and out. The examples are written in Elixir, and the book closes with a set of laws distilled from the successes and failures of real deployments.

[Get the book from the Pragmatic Bookshelf](https://pragprog.com/titles/khpes/real-world-event-sourcing/)

## Programming WebAssembly with Rust

<img class="book-cover" src="/images/covers/programming-webassembly-with-rust.jpg" alt="Cover of Programming WebAssembly with Rust" width="500" height="600">

*Unified Development for Web, Mobile, and Embedded Applications*. The Pragmatic Bookshelf, 2019.

WebAssembly fulfills the long-awaited promise of web technologies: fast, type-safe code that runs in the browser, on embedded devices, or anywhere else. This book teaches WebAssembly from the ground up, starting with the stack machine itself and hand-written text-format modules, before moving on to compiling Rust to WebAssembly and building interoperability with JavaScript. You will build a checkers game in raw WebAssembly, rebuild it in Rust, and create a front end with the Yew framework. The last part of the book leaves the browser entirely, hosting WebAssembly in native applications, running modules on a Raspberry Pi for an IoT project, and finishing with a multiplayer robot combat engine where players upload their own WebAssembly modules to compete.

[Get the book from the Pragmatic Bookshelf](https://pragprog.com/titles/khrust/programming-webassembly-with-rust/)

## Building Microservices with ASP.NET Core

<img class="book-cover" src="/images/covers/building-microservices-with-aspnet-core.jpg" alt="Cover of Building Microservices with ASP.NET Core" width="381" height="500">

*Develop, Test, and Deploy Cross-Platform Services in the Cloud*. O'Reilly Media, 2017.

Nearly every business, regardless of domain, needs software running in the cloud, and microservices provide the agility and reduced time to market that demands. This hands-on guide shows how to create, test, compile, and deploy microservices using the free and open-source ASP.NET Core framework. It covers test-driven and API-first development, creating and consuming backing services such as databases and queues, building services that depend on external data sources, event sourcing, services that consume and are consumed by other services, externalized configuration, and securing ASP.NET Core services and applications for life in the cloud.

[Get the book from O'Reilly](https://www.oreilly.com/library/view/building-microservices-with/9781491961728/)

## Cloud Native Go

<img class="book-cover" src="/images/covers/cloud-native-go.jpg" alt="Cover of Cloud Native Go" width="500" height="648">

*Building Web Applications and Microservices for the Cloud with Go and React*. Sams Publishing, 2016. Co-authored with Dan Nemeth.

This book describes the modern cloud-native application in detail, covering the factors, disciplines, and habits that make rapid and reliable cloud-native development possible, and it introduces Go as a language especially well suited to that kind of work. After a primer on the language, it walks through continuous delivery pipelines, building microservices in Go, working with backing services, creating a data service, event sourcing and CQRS, security in the cloud, and WebSockets. The final chapters build the front end with React and Flux and bring everything together in a full application.

[Get the book from InformIT](https://www.informit.com/store/cloud-native-go-building-web-applications-and-microservices-9780672337796)

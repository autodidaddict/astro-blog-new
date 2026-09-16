---
lang: en
---

I have been writing technical books for a long time. It started out as an itch that I needed to scratch and turned into a full-on rash. The cycle that drives me is _learn -> build -> teach_, which covers book writing. To be able to teach something to someone in simple terms, I need to gain deep understanding of it.

## Technical books

The following are the most recent and relevant books that pertain to my passions and day job of event-sourcing, distributed systems, secure untrusted computing, massive scale, and high throughput.

### Real-World Event Sourcing

<img class="book-cover" src="/images/covers/real-world-event-sourcing.jpg" alt="Cover of Real-World Event Sourcing" width="500" height="600">

*Distribute, Evolve, and Scale Your Elixir Applications*. The Pragmatic Bookshelf, 2025.

Applications use streams of incoming data to create their own realities. This book is about learning to treat data as events and then interpreting those events to solve genuinely hard problems, from the first line of code through to production. It starts with the fundamental building blocks of an event-sourced system: commands, aggregates, projectors, process managers, injectors, and notifiers. From there it moves into the parts that only show up once you have real systems in the field, such as distributing an application across nodes, modeling failure, replaying history, evolving event schemas without breaking what already exists, securing and testing event-sourced systems, and scaling up and out. The examples are written in Elixir, and the book closes with a set of laws distilled from the successes and failures of real deployments.

[Get the book from the Pragmatic Bookshelf](https://pragprog.com/titles/khpes/real-world-event-sourcing/)

### Programming WebAssembly with Rust

<img class="book-cover" src="/images/covers/programming-webassembly-with-rust.jpg" alt="Cover of Programming WebAssembly with Rust" width="500" height="600">

*Unified Development for Web, Mobile, and Embedded Applications*. The Pragmatic Bookshelf, 2019.

WebAssembly fulfills the long-awaited promise of web technologies: fast, type-safe code that runs in the browser, on embedded devices, or anywhere else. This book teaches WebAssembly from the ground up, starting with the stack machine itself and hand-written text-format modules, before moving on to compiling Rust to WebAssembly and building interoperability with JavaScript. You will build a checkers game in raw WebAssembly, rebuild it in Rust, and create a front end with the Yew framework. The last part of the book leaves the browser entirely, hosting WebAssembly in native applications, running modules on a Raspberry Pi for an IoT project, and finishing with a multiplayer robot combat engine where players upload their own WebAssembly modules to compete.

[Get the book from the Pragmatic Bookshelf](https://pragprog.com/titles/khrust/programming-webassembly-with-rust/)

### Building Microservices with ASP.NET Core

<img class="book-cover" src="/images/covers/building-microservices-with-aspnet-core.jpg" alt="Cover of Building Microservices with ASP.NET Core" width="381" height="500">

*Develop, Test, and Deploy Cross-Platform Services in the Cloud*. O'Reilly Media, 2017.

Nearly every business, regardless of domain, needs software running in the cloud, and microservices provide the agility and reduced time to market that demands. This hands-on guide shows how to create, test, compile, and deploy microservices using the free and open-source ASP.NET Core framework. It covers test-driven and API-first development, creating and consuming backing services such as databases and queues, building services that depend on external data sources, event sourcing, services that consume and are consumed by other services, externalized configuration, and securing ASP.NET Core services and applications for life in the cloud.

[Get the book from O'Reilly](https://www.oreilly.com/library/view/building-microservices-with/9781491961728/)

### Cloud Native Go

<img class="book-cover" src="/images/covers/cloud-native-go.jpg" alt="Cover of Cloud Native Go" width="500" height="648">

*Building Web Applications and Microservices for the Cloud with Go and React*. Sams Publishing, 2016. Co-authored with Dan Nemeth.

This book describes the modern cloud-native application in detail, covering the factors, disciplines, and habits that make rapid and reliable cloud-native development possible, and it introduces Go as a language especially well suited to that kind of work. After a primer on the language, it walks through continuous delivery pipelines, building microservices in Go, working with backing services, creating a data service, event sourcing and CQRS, security in the cloud, and WebSockets. The final chapters build the front end with React and Flux and bring everything together in a full application.

[Get the book from InformIT](https://www.informit.com/store/cloud-native-go-building-web-applications-and-microservices-9780672337796)

### Beyond the Twelve-Factor App

<img class="book-cover" src="/images/covers/beyond-the-twelve-factor-app.jpg" alt="Cover of Beyond the Twelve-Factor App" width="400" height="600">

*Exploring the DNA of Highly Scalable, Resilient Cloud Applications*. O'Reilly Media, 2016.

In 2012, Heroku published the Twelve-Factor App, a set of guidelines for building applications that behave well in the cloud. It was an excellent starting point, but technology marched on and some of it needed revisiting. This short report walks through each of the original factors, explains why it matters, and expands the list to fifteen, adding API first, telemetry, and authentication and authorization while reordering and sharpening the rest. The goal is applications that do more than merely function in the cloud: applications that thrive there, that can be deployed and disposed of without ceremony, and that behave predictably at scale.

[Read the report at O'Reilly](https://www.oreilly.com/library/view/beyond-the-twelve-factor/9781492042631/)

## Earlier work

Before the cloud-native era, I spent the better part of a decade writing about Microsoft's .NET stack, most of it for Sams, with the occasional detour into Apple's platforms. These are the better-known titles from that period.

### Sams Teach Yourself Mac OS X Lion App Development in 24 Hours

<img class="book-cover" src="/images/covers/sams-teach-yourself-mac-os-x-lion-app-development.jpg" alt="Cover of Sams Teach Yourself Mac OS X Lion App Development in 24 Hours" width="459" height="600">

Sams Publishing, 2012.

Twenty-four one-hour lessons for building native Mac applications, written as Lion brought iOS ideas such as multitouch gestures, iCloud, and the Mac App Store to the desktop. It starts with Xcode, Objective-C, and the Model-View-Controller pattern, then works through Interface Builder and the Cocoa controls, memory management with automatic reference counting, and Core Data for persistence. The later hours cover gestures and multitouch, iCloud integration and document versioning, Core Animation, web services and drag and drop, and finally submitting to the Mac App Store and adding in-app purchases.

[View the book on InformIT](https://www.informit.com/store/sams-teach-yourself-mac-os-x-lion-app-development-in-9780672335815)

### Windows Phone 7 for iPhone Developers

<img class="book-cover" src="/images/covers/windows-phone-7-for-iphone-developers.jpg" alt="Cover of Windows Phone 7 for iPhone Developers" width="467" height="600">

Addison-Wesley Professional, Developer's Library, 2011.

Written for developers who already knew their way around iOS and wanted to bring their apps to Windows Phone 7. It walks through the entire Windows Phone SDK side by side with Apple's, starting with C# for Objective-C programmers and building interfaces with Silverlight and XAML in Visual Studio and Expression Blend. From there it covers device services such as the accelerometer, GPS, camera, and contacts, then the pieces unique to the platform: Application Tiles, push notifications, the phone execution model, and local storage. It closes with MVVM, unit testing, connected social games, application security, and getting an app through the Marketplace.

[View the book on InformIT](https://www.informit.com/store/windows-phone-7-for-iphone-developers-9780132657747)

### ASP.NET 4 Unleashed

<img class="book-cover" src="/images/covers/aspnet-4-unleashed.jpg" alt="Cover of ASP.NET 4 Unleashed" width="385" height="500">

Sams Publishing, 2010. Co-authored with Stephen Walther and Nate Dudek.

A doorstop of a book, and the comprehensive reference for ASP.NET 4 in its day. It covers the entire platform from the ground up: Web Forms and validation, Master Pages and themes, data-driven applications with the database controls and LINQ to SQL, the Chart control, custom controls and components, membership and authentication, URL routing, caching and performance, AJAX, the ASP.NET MVC framework, and deploying and configuring applications, all backed by hundreds of realistic code examples.

[View the book on InformIT](https://www.informit.com/store/asp.net-4-unleashed-9780672331121)

### Microsoft SharePoint 2007 Development Unleashed

<img class="book-cover" src="/images/covers/microsoft-sharepoint-2007-development-unleashed.jpg" alt="Cover of Microsoft SharePoint 2007 Development Unleashed" width="461" height="600">

Sams Publishing, 2007. Co-authored with Robert Foster.

A guide for .NET developers building enterprise applications on SharePoint 2007 with ASP.NET 2.0 and C#. It covers the SharePoint object model and CAML, Features and Solutions, sites, webs, and document libraries, lists and event handlers, workflows and business data integration, Web Parts, Excel Services and user profiles, the web services for document workspaces and lists, records repositories, and the security and debugging techniques that go with all of it.

[View the book on InformIT](https://www.informit.com/store/microsoft-sharepoint-2007-development-unleashed-9780672329036)

### Microsoft Visual C# 2005 Unleashed

<img class="book-cover" src="/images/covers/microsoft-visual-csharp-2005-unleashed.jpg" alt="Cover of Microsoft Visual C# 2005 Unleashed" width="385" height="500">

Sams Publishing, 2006.

A premium reference for C# 2.0 and the .NET Framework 2.0, written to be read front to back or pulled off the shelf to solve a single problem. It covers the language itself, from expressions and control structures to the workings of the garbage collector, then moves through Windows Forms and ASP.NET user interfaces before spending its back half on the advanced material: code access security, remoting, peer-to-peer applications, smart clients, Enterprise Services, and web services with WSE.

[View the book on InformIT](https://www.informit.com/store/microsoft-visual-c-sharp-2005-unleashed-9780672327766)

### Microsoft Visual C# .NET 2003 Unleashed

<img class="book-cover" src="/images/covers/microsoft-visual-csharp-dotnet-2003-unleashed.jpg" alt="Cover of Microsoft Visual C# .NET 2003 Unleashed" width="524" height="648">

Sams Publishing, 2004. Co-authored with Lonny Kruger.

A comprehensive guide to the .NET Framework with C# as the teaching language. It starts with the Visual Studio .NET IDE, the fundamentals of the language, object-oriented programming, and reflection, then builds outward: Windows Forms and ASP.NET applications, ADO.NET data access, XML and web services, code access security, Enterprise Services and COM interop, and debugging and monitoring applications once they are running. Nearly a thousand pages, from the garbage collector at the bottom to distributed applications at the top.

[View the book on InformIT](https://www.informit.com/store/microsoft-visual-c-sharp-.net-2003-unleashed-9780768665963)

## Fiction

Not everything I write has code in it. The Sigilord Chronicles is an epic fantasy series about Urus Noellor, a boy born deaf into a warrior culture that has no place for him, and the ancient magic he awakens.

### The Fifth Vertex

<img class="book-cover" src="/images/covers/the-fifth-vertex.jpg" alt="Cover of The Fifth Vertex" width="333" height="500">

*The Sigilord Chronicles, Book 1*. 2014.

Urus Noellor, about to be publicly branded a burden to the warrior society of Kest, stands on a rooftop ready to throw himself over the edge. His failed attempt unlocks a form of magic thought to have died out thousands of years before, and it may be the key to stopping an equally ancient enemy. Together with Goodwyn, the greatest warrior in Kest, and Cailix, a mysterious orphan, Urus must stop a powerful group of sorcerers from destroying the five hidden vertices that ward the world against threats from beyond. As the battle spreads to the neighboring realms, Goodwyn confronts the realities of war and death, Cailix uncovers a devastating truth, and Urus discovers his gifts, his courage, and his true identity, experiencing all of it in profound, lonely silence.

[Get the book on Amazon](https://www.amazon.com/Fifth-Vertex-Sigilord-Chronicles/dp/0990647919)

### The Blood Sigil

<img class="book-cover" src="/images/covers/the-blood-sigil.jpg" alt="Cover of The Blood Sigil" width="333" height="500">

*The Sigilord Chronicles, Book 2*. 2015.

Urus has defeated the Order of the Sanguine Crystal, and his reward is a cell. Now a prisoner of the Council of Balance, he awaits a trial that could mean life or death for crimes he didn't know he had committed. Back on Emys, six months have passed. Cailix has grown attached to her new family on Aldsdowne, until the return of Anderis threatens to tear it apart, and Goodwyn and Therren have traveled to the capital city of Niragan seeking help to rebuild Waldron after the battles that nearly destroyed it. All of them are about to be drawn into the rekindling of an ancient feud, one that threatens to unleash an enemy even more powerful than the blood mages.

[Get the book on Amazon](https://www.amazon.com/Blood-Sigil-Sigilord-Chronicles/dp/1546459146)

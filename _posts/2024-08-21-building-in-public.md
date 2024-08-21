---
layout: post
title: Building in Public
date: 2024-08-21 15:12 +0200
categories:
- aurelianix
---
This is an exercise in finishing a project. I've been building a personal accounting app for myself during the last couple of months. There are a lot of unknowns and I will talk about the details of this app in a follow-up post, but first I want to ramble a bit about the things that have been on my chest.

## Building in public

I've decided to start building in public. This means talking about my progress, decisions I take on the way and, I guess most importantly, the thoughts that plague me. I've noticed time and time again that just blurting out what's in my mind helps me focusing on the stuff I want to focus on.

I plan on doing regular blog posts showcasing some of the stuff I'm working on. My intention is not to drive up engagement or hype, but to be able to show and tell my progress so I can be proud. I've got very little time on my hands. I have two toddlers at home and both my wife and I have day jobs that we pursue with passion and energy. This takes a toll, both on my remaining strength to sit down at home and build, but also on my free time which has become almost non-existent.

Publishing regular updates will surely throw me back even more (I get maybe 6 hours of work done in a week), but I'm hopeful that it'll help me stay on track, which brings me directly to my next section.

## Losing focus

I'm a perfectionist. Unfortunately, being a perfectionist often gets in the way. Just ysterday I spent an hour "optimizing" an unused part of my code. I do this all the time and I notice how it sucks the energy out of me.

My app is being built with React Native for simple reasons: A good web story (react-native-web is excellent) and a good native story (I hate webviews as apps on my phone). The other option would probably be Flutter, but on my underpowered Android the web performance is horrinle. Unfortunately, I loathe writing JS/TS (that's a story for another day), so I'd like my business logic to be in a language that is not TypeScript, but also have everything being calculated in the app itself.

I considered several options and ended up with F# which can compile to TypeScript through [Fable](https://fable.io). I actually did a first draft using Rust, but I didn't like how cumbersome it is to integrate into React Native. F# is a great language, but the generated TypeScript code is pretty unreadable, there are often incorrect or incomplete type annotations (the F#-to-TS compiler startet as an F#-to-JS compiler and added types later, so it's still lacking a bit sometimes) especially in libraries and the JS performance has bitten me several times (I'm looking at you, custom hashing function for dictionary lookups).

Today marks the nine month anniversary of me starting my app. I would not choose F# again, for the reasons above, but also because it often feels like a second-class citizen within .NET itself. Rust would probably be perfect if interop with Native Modules in React Native were easier, otherwise I might choose [Gleam](https://gleam.run). Gleam can compile to JS (including TypeScript definitions) and runs on the BEAM VM, so an application built with Elixir+Phoenix, business logic in Gleam seems possible. I've always wanted to get more into Elixir as well, and from toying around, Gleam's generated JavaScript looks pretty readable. Another option would be js_of_ocaml, but it does not generate TS binding, only JS code, and I'd like type definitions included.

I sincerely hope that Rusts GUI story improves to a point where it can be an equal to JavaScript.


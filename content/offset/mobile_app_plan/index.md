+++
title = "Offset mobile app plan"
description = "Initial work on an Offset mobile app"
date = 2020-02-11
+++

## Intro to Offset mobile app plan

Offset is a digital credit card, powered by trust between people.
So far the main development on Offset was focused on designing a reasonable
open protocol, and on developing the core parts of Offset. One thing that was
neglected was a good graphical user interface.

I spent the last month designing `stcompact`: An API bridge between Offset's
core (the Offset node) and what would be a mobile app for Offset.

Why a mobile app? Because a mobile phone is something most people carry with
them everywhere. Having an Offset app should allow to pay or collect payment
anywhere, using your mobile phone.

Offset mobile app is open source (under the AGPL3 license). You can check the
repository [on github](https://github.com/freedomlayer/offset_mobile).

## Preliminary mobile app user flow

I include here a preliminary (low fidelity) user flow diagram for the planned
mobile app. The diagram roughly describes all the possible screens that the
Offset app should have.

You can download the diagram in two formats:

- [svg diagram](offset_mobile.svg).
- [drawio diagram](offset_mobile.drawio).


The user flow diagram is somewhat large, you will probably need to zoom-in and zoom-out to see
everything. Any ideas or suggestion are highly appreciated.

## Other progress on Offset

- Added the [multi-currency feature](https://github.com/freedomlayer/offset/pull/240), allowing to use multiple user-chosen currencies with a single Offset node.
- Changed `MultiCommit` protocol message [to have constant size](https://github.com/freedomlayer/offset/pull/242).
- Make Offset [build on stable rust](https://github.com/freedomlayer/offset/pull/248).
- Simplified the Offset protocol by removing [remote-max-debt](https://github.com/freedomlayer/offset/pull/255) and [enable/disable requests](https://github.com/freedomlayer/offset/pull/256).
- Implemented [built_union.dart](https://github.com/freedomlayer/built_union.dart), to allow serializing and deserializing sum-types between stcompact and Offset mobile app (Written in dart).

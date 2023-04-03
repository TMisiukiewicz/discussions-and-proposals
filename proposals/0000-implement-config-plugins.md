---
title: Implement Expo Config Plugins into React Native CLI
author:
- Tomasz Misiukiewicz
date: today
---

# RFC0000: Supporting Expo Config Plugins in React Native CLI

## Summary

Implementing Expo Config Plugins into React Native CLI to support modifying application's config without having to manually edit the native files.

## Basic example

~~If the proposal involves a new or changed API, include a basic code example. Omit this section if it's not applicable.~~

## Motivation

The main motivation between this proposal is simplifying the process of upgrading apps to the newest React Native version. Very often it requires some additional manual steps on the native side, resolving conflicts etc. With Expo Config Plugins implemented into React Native, it would be possible to store all the native-related config on the JS side and apply it into native files whenever it's needed, e.g while running `react-native upgrade` - without having Expo in the project. On top of that, it would become possible to build config plugins for Windows and Mac.

## Detailed design

~~This is the bulk of the RFC. Explain the design in enough detail for somebody familiar with React Native to understand, and for somebody familiar with the implementation to implement. This should get into specifics and corner-cases, and include examples of how the feature is used. Any new terminology should be defined here.~~

## Drawbacks

~~Why should we _not_ do this? Please consider:~~

- ~~implementation cost, both in term of code size and complexity~~
- ~~whether the proposed feature can be implemented in user space~~
- ~~the impact on teaching people React Native~~
- ~~integration of this feature with other existing and planned features~~
- ~~cost of migrating existing React Native applications (is it a breaking change?)~~

~~There are tradeoffs to choosing any path. Attempt to identify them here.~~

## Alternatives

~~What other designs have been considered? Why did you select your approach?~~

## Adoption strategy

~~If we implement this proposal, how will existing React Native developers adopt it? Is this a breaking change? Can we write a codemod? Should we coordinate with other projects or libraries?~~

## How we teach this

~~What names and terminology work best for these concepts and why? How is this idea best presented? As a continuation of existing React patterns?~~

~~Would the acceptance of this proposal mean the React Native documentation must be re-organized or altered? Does it change how React Native is taught to new developers at any level?~~

~~How should this feature be taught to existing React Native developers?~~

## Unresolved questions

~~Optional, but suggested for first drafts. What parts of the design are still TBD?~~

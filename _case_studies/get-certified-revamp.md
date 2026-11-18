---
title: Get Certified - Revamp
summary: Modernizing and reimagining old app implementation
hero: /assets/images/hero-nook.svg
platforms: [iOS]
stack: [SwiftUI, UI/UX]
order: 5
featured: false
before_shots:
  - src: /assets/images/get-certified/before/app-icon.png
    caption: App icon not clipped correctly
  - src: /assets/images/get-certified/before/launch-screen.png
    caption: Default launch screen from the base project
  - src: /assets/images/get-certified/before/not-full-screen.png
    caption: Not full screen
  - src: /assets/images/get-certified/before/home-button-navigation.png
    caption: Unconventional navigation with the Home button
  - src: /assets/images/get-certified/before/catalog-browsing.png
    caption: Dated browsing of a catalog
  - src: /assets/images/get-certified/before/switch-footer.png
    caption: Switch usage, inconvenient footer
  - src: /assets/images/get-certified/before/text-truncation.png
    caption: Text truncation
  - src: /assets/images/get-certified/before/uncentered-loading.png
    caption: Uncentered loading indicator
  - src: /assets/images/get-certified/before/inquiry-form.png
    caption: "Weird UX: sending an inquiry only asks for personal details"
---

Get Certified was the first app I ever built. I wrote it in 2014 with what was then the standard iOS stack: Objective-C, UIKit, storyboards and XIBs, all in Xcode. After it shipped it never got another update, and the platform has moved on without it.

I was proud of it back then. Looking at it now, it's a little embarrassing to bring up. So I'm using this case study to fix that: go back to the app, look at what has aged, and rebuild it so it meets modern app standards.

## The original app

Get Certified is Power Mac Center's app for its Apple Authorized Training Center (AATC). It lets you browse the catalog of training courses, put together a training list, see schedules, and contact the center during business hours.

## Findings

{% include shots.html items=page.before_shots %}

## The revamp

I'm rebuilding the app in SwiftUI instead of patching the old Objective-C project. The plan comes straight from the findings: every change below fixes something on that list.

### 1. A new app icon

The old icon was clipped incorrectly by the system. The new one is designed for the current iOS icon grid and mask, so it sits correctly on the Home Screen and adapts to the system's light, dark, and tinted appearances.

### 2. A real launch screen, edge to edge

The app still opened on the default launch screen from the base project, and it didn't fill the display. I'm replacing it with a branded launch screen and making the app full screen, edge to edge, with content laid out within the safe areas.

### 3. Conventional navigation

Getting around depended on a Home button, which isn't how iOS apps work. The revamp uses standard push navigation throughout: you drill into a screen, and the back button or a swipe takes you back to where you came from.

### 4. Reworking the key screens

This is the biggest part of the work:

- **Home**: a proper starting point for the app.
- **Search**: find a course directly instead of browsing for it.
- **Catalog listing**: a modern list that replaces the dated catalog browsing.
- **The small things**: no more truncated text, a centered loading indicator, no awkward switch and footer, and consistent alignment and spacing across every screen.

## Results

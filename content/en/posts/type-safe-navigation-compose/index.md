---
date: '2025-05-21T16:00:28+08:00'
draft: true
title: "Type-Safe Navigation in Jetpack Compose"
series: ["deep-link-compose-fcm"]
series_order: 1  # 2, 3
tags: ["android", "compose", "navigation", "deep-links", "fcm"]
categories: ["Android Development"]
---

## What You'll Learn
- Setting up type-safe navigation with Kotlin Serialization
- Creating serializable route classes
- Navigation between screens
- Benefits over string-based navigation

## Topics
- @Serializable data classes for routes
- NavHost setup with type-safe routes
- Navigation with arguments
- Best practices and gotchas
- 
## Library Requirement
- Navigation Compose version >= 2.8.0 and version < 3.0.0 (Updated June 2025, Google have released a new Navigation3 Library for Jetpack Compose which is now in alpha with better backstack handling and some breaking changes.)
- KotlinX.Serialization
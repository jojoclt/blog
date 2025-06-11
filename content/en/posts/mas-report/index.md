---
date: '2025-06-11T19:55:28+08:00'
draft: false
title: "Get All gradle Package Dependencies for MAS Analysis"
series_order: 1  # 2, 3
tags: ["android", "compose", "build", "gradle"]
categories: ["Android Development"]
---


| ![space-1.jpg](gradle.png) |
| :------------------------: |
|     *Version*                |
As we migrate to version catalog, it is now easier than ever to share the package depencies between submodules, but it is now hard to extract those data back for the Mobile Application Security Package Analysis.

But there is actually an easy way to do that, which can be achieve by a few clicks in Android Studio IDE.

You can run `gradle :app:androidDependencies` in Android Studio to get all the compiled dependencies use in the project, this will listed all the dependencies graph under that aar packages.
![alt text](gradle_dep.png)

Or it can be get from `File` > `Project Structure` > `Dependencies` 
![alt text](project_structure.png)

On the right hand side we can see the `Resolved Dependencies` and each build flavors (More on Build Variants here...).

For example I will choose `devPermRelease`

![Right Tab Enlarged](image.png)

You can see all the resolved dependencies here, you can even highlight all the dependencies in that group and use *Copy Shortcuts* on your Operating System (Right Click doesn't seemed to work here) and then paste it in your favourite text editor.

![Copied Dependencies Example](image-1.png)


### Key Takeaways

Learned how to get the dependencies list on android exported in a text format for report usages.
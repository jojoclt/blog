---
date: '2024-12-21T16:00:28+08:00'
draft: true
title: '🚧Integrating Deep Links with Type-Safe Navigation Compose (and custom parsing) and fcm'
series: ["deep-link-compose-fcm"]
seriesOrder: 2  # 2, 3
tags: ["android", "compose", "navigation", "deep-links", "fcm"]
categories: ["Android Development"]
# ShowToc: true
# TocOpen: true
---

> ## This Post is being built
By using deep links, you can jump from the notifications to a certain routes in NavGraph or even Nested Graph

## Prerequisites
- KotlinX.Serialization
- Navigation Compose
- Navigating Deep Links with Intents

## Url Parameters
![image info](../res/url_scheme.png)

## Using Something

The following `DetailPage` class is used as an example for the navigation
```kotlin
@Serializable
data class DetailPage(
    @SerialName("title_name")
    val title: String,
    @SerialName("sub_title")
    val subTitle: String? = null
    
    val id: Int,
)
```


> If you want the `navDeepLink` to parse your URI, you should put it in this format 
> 
> `appName://DetailPage/{title_name}/{id}?sub_title={sub_title}`


In the compose navigation the code will look like

```kotlin
composable<DetailPage>(
    deepLinks = listOf<DetailPage>(
        basePath = "appName://DetailPage",
    )
)

```
 
### What if the Uri is in different format?

For example, you want to extract the id and name pattern from the uri or using custom uri altogether. together, you might want to write a custom `uriPattern` in the NavDeepLink.navDeepLink API

<!-- It looks difficult to build DeepLinks such as example://full_name/{first_name}/{last_name}/edit -->


It can be put together as
```kotlin {hl_lines=[5]}
composable<DetailPage>(
    deepLinks = listOf<DetailPage>(
        basePath = "appName://DetailPage",
    ) {
        uriPattern =  "appName://{id}_{title_name}?sub_title={sub_title}"
    }
) {
    val args = it.toRoute<DetailPage>
}
```
Then args can be retrieve using 

```kotlin {linenos=table}
Or even create something difficult to build like
composable<DetailPage>(
    deepLinks = listOf<DetailPage>(
        basePath = "appName://DetailPage",
    ) {
        uriPattern =  "example://full_name/{first_name}/{last_name}/edit"
    }
)


```

> [!NOTE]
> This uriPattern might not be type safe, but it offers flexibility for those in need

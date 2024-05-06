5/1/24

# Safe Args in Navigation Compose!

Version: [2.8.0-alpha08](https://developer.android.com/jetpack/androidx/releases/navigation#2.8.0-alpha08 "AndroidX Releases for Navigation")

In the latest update of Navigation for Compose, they introduced Safe Args using KotlinX Serialization! And it is
amazing!
I already spent the time to convert one of my side projects, [OtakuWorld](https://github.com/jakepurple13/OtakuWorld/),
into using it.
It makes the code much more readable and so fun and easy to use!

## Contents

* [Quick Overview](#quick-overview)
* [So why write about it?](#so-why-write-about-it)
* [The Problem](#the-problem)
* [The Solution](#the-solution)
  * [Deep Link](#deep-link)
  * [Update (5/6/24)](#update-5-6-24)
* [Conclusion](#conclusion)

## Quick Overview

A quick overview of what this brought (or
read [this great article](https://medium.com/androiddevelopers/navigation-compose-meet-type-safety-e081fb3cf2f8) for a
quick summary):

Instead of needing to have all of your routes as strings like:

```Kotlin
sealed class Screen(val route: String) {
    data object ImportListScreen : Screen("import_list") {
        fun navigate(navController: NavController, uri: Uri) {
            navController.navigate("$route?uri=$uri") { launchSingleTop = true }
        }
    }
    
    data object GlobalSearchScreen : Screen("global_search") {
        fun navigate(navController: NavController, title: String? = null) {
            navController.navigate("$route?searchFor=$title") { launchSingleTop = true }
    }
    
    data object OtherSettings : Screen("others_settings")
    
    data object DetailsScreen : Screen("details")
}

composable(
    Screen.ImportListScreen.route + "?uri={uri}"
) { ImportListScreen() }

composable(
    Screen.GlobalSearchScreen.route + "?searchFor={searchFor}",
    arguments = listOf(navArgument("searchFor") { nullable = true }),
    enterTransition = { slideIntoContainer(AnimatedContentTransitionScope.SlideDirection.Up) },
    exitTransition = { slideOutOfContainer(AnimatedContentTransitionScope.SlideDirection.Down) }
) {
    GlobalSearchView(
        notificationLogo = notificationLogo,
        isHorizontal = windowSize.widthSizeClass == WindowWidthSizeClass.Expanded
    )
}

composable(
    Screen.DetailsScreen.route + "/{model}",
    deepLinks = listOf(
        navDeepLink {
            uriPattern = genericInfo.deepLinkUri + "${Screen.DetailsScreen.route}/{model}"
        }
    ),
    enterTransition = { slideIntoContainer(AnimatedContentTransitionScope.SlideDirection.Up) },
    exitTransition = { slideOutOfContainer(AnimatedContentTransitionScope.SlideDirection.Down) }
) {
    DetailsScreen(
        logo = notificationLogo,
        windowSize = windowSize
    )
}

composable(
    Screen.OtherSettings.route,
    enterTransition = { slideIntoContainer(AnimatedContentTransitionScope.SlideDirection.Start) },
    exitTransition = { slideOutOfContainer(AnimatedContentTransitionScope.SlideDirection.End) },
) { PlaySettings() }

//Then to navigate

Screen.GlobalSearchScreen.navigate(navController, info.title)
navController.navigate(Screen.OtherSettings.route)

```

You can do things like:

```Kotlin
@Serializable
sealed class Screen(val route: String) {
    @Serializable
    data class ImportListScreen(val uri: String) : Screen("import_list")
    
    @Serializable
    data class GlobalSearchScreen(
        val title: String? = null,
    ) : Screen("global_search")
    
    @Serializable
    data object OtherSettings : Screen("others_settings")
}

composable<Screen.ImportListScreen> { ImportListScreen(it.toRoute()) }

composable<Screen.GlobalSearchScreen>(
    enterTransition = { slideIntoContainer(AnimatedContentTransitionScope.SlideDirection.Up) },
    exitTransition = { slideOutOfContainer(AnimatedContentTransitionScope.SlideDirection.Down) }
) {
    GlobalSearchView(
        globalSearchScreen = it.toRoute<Screen.GlobalSearchScreen>(),
        notificationLogo = notificationLogo,
        isHorizontal = windowSize.widthSizeClass == WindowWidthSizeClass.Expanded
    )
}

composable<Screen.OtherSettings>(
    enterTransition = { slideIntoContainer(AnimatedContentTransitionScope.SlideDirection.Start) },
    exitTransition = { slideOutOfContainer(AnimatedContentTransitionScope.SlideDirection.End) },
) { PlaySettings() }

//Then to navigate

navController.navigate(Screen.GlobalSearchScreen(info.title))
navController.navigate(Screen.OtherSettings)
```

It looks so much cleaner! You call `navBackStackEntry.toRoute<T>()` to get the class!
It is wonderful!

## So why write about it?

So, while I was playing around and setting it up, I ran into a problem...<i>Deep Linking</i>.
In many apps with notifications, you usually want to navigate to a certain screen from that notification.
The problem I ran into is that in my OtakuWorld project, my notifications direct you to the details screen from the
notifications, or the NotificationScreen if the group was pressed.

My Notifications destinations:

```Kotlin
data object NotificationScreen : Screen("notifications")
data object DetailsScreen : Screen("details")
```

To create a deep link with navigation compose:
```Kotlin
composable(
    route = Screen.NotificationScreen.route,
    deepLinks = listOf(
        navDeepLink { 
            uriPattern = "appInfo:// + Screen.NotificationScreen.route 
        }
    ),
    enterTransition = { slideIntoContainer(AnimatedContentTransitionScope.SlideDirection.Up) },
    exitTransition = { slideOutOfContainer(AnimatedContentTransitionScope.SlideDirection.Down) }
) { ... }
```

Pretty simple, right?. And to set the notification's destination to go there, you can create an intent like:
```Kotlin
val deepLinkIntent = Intent(
    Intent.ACTION_VIEW,
    "$deepLinkUri${Screen.NotificationScreen.route}".toUri(),
    context,
    MainActivity::class.java
)
```

## The Problem

Now that we are on the same page, here is where my problem comes in. The notification screen was easy and required ZERO
changes. It just worked...
The details screen, however, which now has arguments:

```Kotlin
@Serializable
data object DetailsScreen : Screen("details") {
    @Serializable
    data class Details(
        val title: String,
        val description: String,
        val url: String,
        val imageUrl: String,
        val source: String,
    )
}
```

brought a set of problems I wasn't prepared for. But being unprepared has never stopped me before, so I started trying
things out.

Originally, my uri for the details intent was:

```Kotlin
"$deepLinkUri${Screen.DetailsScreen.route}/${Uri.encode(itemModel.toJson(ApiService::class.java to ApiServiceSerializer()))}".toUri()
```

Long, I know, but it did work.

So, what's the issue? Well, My `Screen.DetailsScreen.Details` is not an object...It's a class.
So how can I set it up so that I can navigate to this screen which I have set up with the amazing Safe Args?

I can't just `Screen.DetailsScreen.Details.toString()` and call it a day. It does compile since it calls the companion
object, but that's about it.
Great...What about creating the path myself? Like so:

```Kotlin
val addon = "&title=${itemModel?.title}&description=${itemModel?.description}&imageUrl=${itemModel?.imageUrl}&source=${itemModel?.source}&url=${itemModel?.url}"
```

I was trying a whole bunch of things, just throwing things at the wall to see if anything stuck.
Going down deep into the navigation compose source code to see if it could give me any insights or ideas.

Nope, it still did not work. At this point, I was stumped. So, I kept Details using strings as routes and called it a
day...
But if you know me, you know I work on something until I've truly exhausted all options.

Plus, it'd be a pretty lame post if I didn't include an answer, right?

## The Solution

As I was trying to figure this out, I found a pretty interesting function (that I purposely left out of the previous
section):

```Kotlin
@OptIn(InternalSerializationApi::class)
@RestrictTo(RestrictTo.Scope.LIBRARY_GROUP)
public fun <T : Any> T.generateRouteWithArgs(
    typeMap: Map<String, NavType<Any?>>
): String = RouteEncoder(this::class.serializer(), typeMap).encodeRouteWithArgs(this)
```

Coming straight from `androidx.navigation.serialization`. Okay, that looks neat...Whaaaat does it do?
Well, exactly what it looks like! It generates a route with arguments!

```Kotlin
com.programmersbox.uiviews.utils.Screen.DetailsScreen.Details/TitleHere/DescriptionHere/UrlHere/ImageUrlHere/SourceHere
```

That's it! That is exactly what I need! That gives the exact destination I want...WITH the arguments! Great! Ship it!

...

Okay, so it wasn't as easy as just calling:

```Kotlin
val route = Screen.DetailsScreen.Details(
    title = itemModel?.title ?: "",
    description = itemModel?.description ?: "",
    url = itemModel?.url ?: "",
    imageUrl = itemModel?.imageUrl ?: "",
    source = itemModel?.source?.serviceName ?: "",
).generateRouteWithArgs(emptyMap())
```

I actually ran into issues with just that. It couldn't tell the type of the parameters.
Okay, that's easy to deal with. That `emptyMap` is looking for the parameter names to what `NavType` they are.
Great! We deal with that when creating the destination! So let me just fill it in:

```Kotlin
val route = Screen.DetailsScreen.Details(
    title = itemModel?.title ?: "",
    description = itemModel?.description ?: "",
    url = itemModel?.url ?: "",
    imageUrl = itemModel?.imageUrl ?: "",
    source = itemModel?.source?.serviceName ?: "",
).generateRouteWithArgs(
    mapOf(
        "title" to NavType.StringType,
        "description" to NavType.StringType,
        "url" to NavType.StringType,
        "imageUrl" to NavType.StringType,
        "source" to NavType.StringType,
    )
)
```

Easy! Okay and...why is it giving me an error?

```
Type mismatch.
    Required:
        NavType<Any?>
    Found:
      NavType<String?>
Type mismatch.
    Required:
        Pair<String, NavType<Any?>>
    Found:
        Pair<String, NavType<String?>>
```

Wat? But it's extending NavType?? Why won't it just-...You know what?:

```Kotlin
val route = Screen.DetailsScreen.Details(
    title = itemModel?.title ?: "",
    description = itemModel?.description ?: "",
    url = itemModel?.url ?: "",
    imageUrl = itemModel?.imageUrl ?: "",
    source = itemModel?.source?.serviceName ?: "",
).generateRouteWithArgs(
    mapOf(
        "title" to NavType.StringType as NavType<Any?>,
        "description" to NavType.StringType as NavType<Any?>,
        "url" to NavType.StringType as NavType<Any?>,
        "imageUrl" to NavType.StringType as NavType<Any?>,
        "source" to NavType.StringType as NavType<Any?>,
    )
)
```

Done! No errors! Run!

...Didn't work...What now?

Well, now that this is all set up, there is one last thing to deal with...Creating the deep link for the destination.

### Deep Link

Remember how I mentioned:
`Well, My Screen.DetailsScreen.Details is not an object...It's a class.`
and
`I can't just use Screen.DetailsScreen.Details.toString() and call it a day.`
?

Well, my words are back. And we can't use `generateRouteWithArgs` this time since that returns a route, not
a `uriPattern`.

During my search when I found `generateRouteWithArgs`, I found another real interesting function:

```Kotlin
internal fun <T> KSerializer<T>.generateRoutePattern(
    typeMap: Map<KType, NavType<*>> = emptyMap(),
    path: String? = null,
): String {
    assertNotAbstractClass {
        throw IllegalArgumentException(
            "Cannot generate route pattern from polymorphic class " +
                "${descriptor.capturedKClass?.simpleName}. Routes can only be generated from " +
                "concrete classes or objects."
        )
    }

    val map = mutableMapOf<String, NavType<Any?>>()
    for (i in 0 until descriptor.elementsCount) {
        val argName = descriptor.getElementName(i)
        val type = descriptor.getElementDescriptor(i).computeNavType(argName, typeMap)
        map[argName] = type
    }
    val builder = if (path != null) {
        RouteBuilder.Pattern(path, this, map)
    } else {
        RouteBuilder.Pattern(this, map)
    }
    for (elementIndex in 0 until descriptor.elementsCount) {
        builder.addArg(elementIndex)
    }
    return builder.build()
}
```

Ooh! Generate Route Pattern?? But it's internal...Okay, I can just copy and paste the code, right?
Not really. That `RouteBuilder` is also internal. I'm all for copy/pasting code, but there's a limit.

So, what do I do? Well, remember when I showed exactly what `generateRouteWithArgs` returns? We can use that knowledge
to create a pattern!

```Kotlin
uriPattern = genericInfo.deepLinkUri + "com.programmersbox.uiviews.utils.Screen.DetailsScreen.Details/{title}/{description}/{url}/{imageUrl}/{source}"
```

And guess what? It worked. It works EXACTLY as it should!

```Kotlin
composable<Screen.DetailsScreen.Details>(
    deepLinks = listOf(
        navDeepLink {
            uriPattern = genericInfo.deepLinkUri +
                    "com.programmersbox.uiviews.utils.Screen.DetailsScreen.Details/{title}/{description}/{url}/{imageUrl}/{source}"
        }
    ),
    enterTransition = { slideIntoContainer(AnimatedContentTransitionScope.SlideDirection.Up) },
    exitTransition = { slideOutOfContainer(AnimatedContentTransitionScope.SlideDirection.Down) }
) {
    DetailsScreen(
        detailInfo = it.toRoute(),
        logo = notificationLogo,
        windowSize = windowSize
    )
}
```

### Update (5/6/24)

I have since found a way to generate the `navDeepLink` using a built-in method!

```Kotlin
public inline fun <reified T : Any> navDeepLink(
    basePath: String,
    typeMap: Map<KType, @JvmSuppressWildcards NavType<*>> = emptyMap(),
    noinline deepLinkBuilder: NavDeepLinkDslBuilder.() -> Unit = { }
): NavDeepLink = navDeepLink(basePath, T::class, typeMap, deepLinkBuilder)
```

This does exactly what we want!
With a little needed help.
If I just do:

```Kotlin
navDeepLink<Screen.DetailsScreen.Details>("")
```

That's an error since digging down, the basePath cannot be empty.
So!
If I do:

```Kotlin
navDeepLink<Screen.DetailsScreen.Details>(
    basePath = genericInfo.deepLinkUri + Screen.DetailsScreen.Details::class.qualifiedName
)
```

It works perfectly! The output becomes:

```
genericInfo.deepLinkUri + "com.programmersbox.uiviews.utils.Screen.DetailsScreen.Details/{title}/{description}/{url}/{imageUrl}/{source}"
```

This is still not perfect as I did still need to add `Screen.DetailsScreen.Details::class.qualifiedName`
to be the basePath,
but it is certainly better than manually typing it! And we take those!

### End of Update

It was perfect!
I ran it, and it was working.........Until I tried to pass an empty string through it, and it crashed.
This confused me for longer than I'm willing to admit.
The generated string looked like:

```
com.programmersbox.uiviews.utils.Screen.DetailsScreen.Details/TitleHere//UrlHere/ImageUrlHere/SourceHere
```

Notice the issue?
`TitleHere//UrlHere`.
It couldn't understand the empty string.
FORTUNATELY, that is a straightforward solve:

```Kotlin
val route = Screen.DetailsScreen.Details(
    title = itemModel?.title.orEmpty().ifEmpty { "NA" },
    description = itemModel?.description.orEmpty().ifEmpty { "NA" },
    url = itemModel?.url.orEmpty().ifEmpty { "NA" },
    imageUrl = itemModel?.imageUrl.orEmpty().ifEmpty { "NA" },
    source = itemModel?.source?.serviceName.orEmpty().ifEmpty { "NA" },
).generateRouteWithArgs(
    mapOf(
        "title" to NavType.StringType as NavType<Any?>,
        "description" to NavType.StringType as NavType<Any?>,
        "url" to NavType.StringType as NavType<Any?>,
        "imageUrl" to NavType.StringType as NavType<Any?>,
        "source" to NavType.StringType as NavType<Any?>,
    )
)
```

## Conclusion

This was a real interesting problem that I'm sure people are going to run into.
Since Safe Args for Navigation Compose JUST came out (at the time of writing this), there are a lot of people that don't
know some of these ins and outs and only what has been documented with the two or three articles posted about it so far.
It was really fun to find this solution, and I hope in one of the next updates; we are given more support for some of
these use cases.

I hope you enjoyed my findings, rantings, ramblings since I sure had a lot of fun dealing with this.

Happy Composing!
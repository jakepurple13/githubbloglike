# Shared Element Transitions

This is something people have been waiting for...well, probably since Compose started. The ability to do Shared Element
Transitions!

There are already a few articles on it by amazing people like [Skydoves][skydoves] and even
the [official documentation](https://developer.android.com/develop/ui/compose/animation/shared-elements)
showing off how to use it...However, right now, I've been seeing it as a little cumbersome to use.
You have to pass scopes all over the place! Which, also, makes previews hard to use!

SO! I came up with a way I'm pretty happy with to handle this.
It might be a little roundabout, but there's no need to be passing scopes down and using context-receivers and what not.

I already spent the time to play around with them in one of my side
projects, [OtakuWorld](https://github.com/jakepurple13/OtakuWorld/).

## Contents

* [Shared Element...Transitions?](#shared-element-transitions)
* [The Problem](#the-problem)
* [The Solution](#my-solution)
* [Conclusion](#conclusion)

## Shared Element...Transitions?

Let's start with discussing this a bit. If you haven't read the articles/documentation above, I'll do a really quick
example.

These allow you to animate an element from one screen to the next to have a more seamless experience.

![From Skydoves article](https://miro.medium.com/v2/resize:fit:640/format:webp/1*YACtgRhSLW3hYGgfYHLgSw.gif)

From [Skydoves'][skydoves] article

It's incredible seeing it in action!

Setting it up goes like this:

```Kotlin
val navController = rememberNavController()
SharedTransitionLayout {
    NavHost(navController, startDestination = "first") {
        composable(
            "first",
            enterTransition = { slideInHorizontally(initialOffsetX = { -it }) },
            exitTransition = { slideOutHorizontally(targetOffsetX = { it }) }
        ) {
            Column {
                TopAppBar(
                    title = { Text("Text") },
                    modifier = Modifier.sharedElement(
                        rememberSharedContentState(key = "appBar"),
                        this@composable,
                    )
                )
                Text(
                    "Lorem ipsum dolor sit amet, consectetur adipiscing elit. Praesent" +
                            " fringilla mollis efficitur. Maecenas sit amet urna eu urna blandit" +
                            " suscipit efficitur eget mauris. Nullam eget aliquet ligula. Nunc" +
                            "id euismod elit. Morbi aliquam enim eros, eget consequat" +
                            " dolor consequat id. Quisque elementum faucibus congue. Curabitur" +
                            " mollis aliquet turpis, ut pellentesque justo eleifend nec.\n",
                )
                Button(onClick = { navController.navigate("second") }) {
                    Text("Navigate to Cat")
                }
            }
        }
        composable(
            "second",
            enterTransition = { slideInHorizontally(initialOffsetX = { -it }) },
            exitTransition = { slideOutHorizontally(targetOffsetX = { it }) }
        ) {
            Column {
                TopAppBar(
                    title = { Text("Cat") },
                    modifier = Modifier.sharedElement(
                        rememberSharedContentState(key = "appBar"),
                        this@composable,
                    )
                )
                Image(
                    Icons.Default.Build,
                    contentDescription = "cute cat",
                    contentScale = ContentScale.FillHeight,
                    modifier = Modifier.clip(shape = RoundedCornerShape(20.dp))
                )
                Button(onClick = { navController.navigate("third") }) {
                    Text("Navigate to Empty Page")
                }
            }
        }
        composable(
            "third",
            enterTransition = { slideInHorizontally(initialOffsetX = { -it }) },
            exitTransition = { slideOutHorizontally(targetOffsetX = { it }) }
        ) {
            Column(Modifier.fillMaxWidth()) {
                Text("Nothing to see here. Move on.")
                Spacer(Modifier.size(200.dp))
                Button(onClick = { navController.popBackStack("first", false) }) {
                    Text("Pop back to Text")
                }
            }
        }
    }
}
```

And there you go! Simple right?

## The Problem

No...Not at all.

See, the annoyances come from `Modifier.sharedElement`, which requires, to be in the `SharedTransitionScope`.
That's not too difficult. You can just make the screen/composable an extension function, right?

Noooot, really. The other part of that same function is one of the parameters...An `AnimatedVisibilityScope`.

Which makes this difficult. ESPECIALLY if you have a large multi-module like project with lots of screens.

You have to pass BOTH the `AnimatedVisibilityScope` AND `SharedTransitionScope` down to whatever component you want to
animate...
TWICE! Since you also need the receiving end too!

If we look at Skydoves' [Pokedex codebase](https://github.com/skydoves/pokedex-compose):

```Kotlin
@Composable
fun SharedTransitionScope.PokedexHome(
    animatedVisibilityScope: AnimatedVisibilityScope,
    homeViewModel: HomeViewModel = hiltViewModel(),
) {
    ...
}

@Composable
fun SharedTransitionScope.PokedexDetails(
    animatedVisibilityScope: AnimatedVisibilityScope,
    detailsViewModel: DetailsViewModel = hiltViewModel(),
) {
    ...
}
```

Which is indeed a way to do it...For a small project with two screens, yeah, it works.

But for projects with...20 screens each with lots of smaller components, it becomes a nightmare of passing things
around, changing things if you don't need certain features, etc.

## My Solution

I had been thinking about how to handle this since SharedElementTransitions came out, and I *think* I have a good
solution.
It isn't the best, or probably the recommended way to do it, but it works and works VERY well.

I make use of Local Compositions!

```Kotlin
@OptIn(ExperimentalSharedTransitionApi::class)
val LocalSharedElementScope = staticCompositionLocalOf<SharedTransitionScope> { error("") }
val LocalNavigationAnimatedScope = staticCompositionLocalOf<AnimatedVisibilityScope?> { null }
```

And I created my own extension function of the new safe args navigation composable:

```Kotlin
public inline fun <reified T : Any> NavGraphBuilder.animatedScopeComposable(
    typeMap: Map<KType, @JvmSuppressWildcards NavType<*>> = emptyMap(),
    deepLinks: List<NavDeepLink> = emptyList(),
    noinline enterTransition: (AnimatedContentTransitionScope<NavBackStackEntry>.() ->
    @JvmSuppressWildcards EnterTransition?)? = null,
    noinline exitTransition: (AnimatedContentTransitionScope<NavBackStackEntry>.() ->
    @JvmSuppressWildcards ExitTransition?)? = null,
    noinline popEnterTransition: (AnimatedContentTransitionScope<NavBackStackEntry>.() ->
    @JvmSuppressWildcards EnterTransition?)? = enterTransition,
    noinline popExitTransition: (AnimatedContentTransitionScope<NavBackStackEntry>.() ->
    @JvmSuppressWildcards ExitTransition?)? = exitTransition,
    noinline sizeTransform: (AnimatedContentTransitionScope<NavBackStackEntry>.() ->
    @JvmSuppressWildcards SizeTransform?)? = null,
    noinline content: @Composable AnimatedContentScope.(NavBackStackEntry) -> Unit,
) = composable<T>(
    typeMap = typeMap,
    deepLinks = deepLinks,
    enterTransition = enterTransition,
    exitTransition = exitTransition,
    popEnterTransition = popEnterTransition,
    popExitTransition = popExitTransition,
    sizeTransform = sizeTransform,
) {
    CompositionLocalProvider(LocalNavigationAnimatedScope provides this) { content(it) }
}
```

And finally, I created my own Modifier that checks if we are in needed scopes and applies:

```Kotlin
@OptIn(ExperimentalSharedTransitionApi::class)
fun Modifier.customSharedElement(key: Any?) = composed {
        val scope = LocalSharedElementScope.current
        val animatedScope = LocalNavigationAnimatedScope.current

        if (animatedScope != null && key != null) {
            with(scope) {
                sharedElement(
                    rememberSharedContentState(key),
                    animatedScope,
                )
            }
        } else {
            this
        }
    }
```

And that's it! This allows me to set things up much easier:

```Kotlin
@Composable
fun Screens() {
    val navController = rememberNavController()
    SharedTransitionLayout {
        NavHost(navController, startDestination = "first") {
            //This could easily be modified to accept the string route
            animatedScopeComposable(
                "first",
                enterTransition = { slideInHorizontally(initialOffsetX = { -it }) },
                exitTransition = { slideOutHorizontally(targetOffsetX = { it }) }
            ) {
                FirstScreen()
            }
            //This could easily be modified to accept the string route
            animatedScopeComposable(
                "second",
                enterTransition = { slideInHorizontally(initialOffsetX = { -it }) },
                exitTransition = { slideOutHorizontally(targetOffsetX = { it }) }
            ) {
                SecondScreen()
            }
            composable(
                "third",
                enterTransition = { slideInHorizontally(initialOffsetX = { -it }) },
                exitTransition = { slideOutHorizontally(targetOffsetX = { it }) }
            ) {
                Column(Modifier.fillMaxWidth()) {
                    Text("Nothing to see here. Move on.")
                    Spacer(Modifier.size(200.dp))
                    Button(onClick = { navController.popBackStack("first", false) }) {
                        Text("Pop back to Text")
                    }
                }
            }
        }
    }
}

@Composable
fun FirstScreen() {
    Column {
        TopAppBar(
            title = { Text("Text") },
            modifier = Modifier.customSharedElement("appBar")
        )
        Text(
            "Lorem ipsum dolor sit amet, consectetur adipiscing elit. Praesent" +
                    " fringilla mollis efficitur. Maecenas sit amet urna eu urna blandit" +
                    " suscipit efficitur eget mauris. Nullam eget aliquet ligula. Nunc" +
                    "id euismod elit. Morbi aliquam enim eros, eget consequat" +
                    " dolor consequat id. Quisque elementum faucibus congue. Curabitur" +
                    " mollis aliquet turpis, ut pellentesque justo eleifend nec.\n",
        )
        Button(onClick = { navController.navigate("second") }) {
            Text("Navigate to Cat")
        }
    }
}

@Composable
fun SecondScreen() {
    Column {
        TopAppBar(
            title = { Text("Cat") },
            modifier = Modifier.customSharedElement("appBar")
        )
        Image(
            Icons.Default.Build,
            contentDescription = "cute cat",
            contentScale = ContentScale.FillHeight,
            modifier = Modifier.clip(shape = RoundedCornerShape(20.dp))
        )
        Button(onClick = { navController.navigate("third") }) {
            Text("Navigate to Empty Page")
        }
    }
}
```

It becomes MUCH easier to just place the `Modifier.customSharedElement` now and no need to pass the same two parameters
20 places.

## Conclusion

This is just my solution, and I'm sure it's not the best, but it makes it the most maintainable for me.

I'd love to hear other people's solutions to this and post them
in [The Discussions Tab](https://github.com/jakepurple13/githubbloglike/discussions) on the repo.
I'm incredibly curious as to what people come up with.


[skydoves]: https://proandroiddev.com/shared-element-transition-in-jetpack-compose-provide-enriched-user-experiences-163d4e435869 "Skydoves' Amazing Article!"
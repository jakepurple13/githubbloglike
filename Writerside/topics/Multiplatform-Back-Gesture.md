# Multiplatform Back Gesture

I ran into a very interesting issue the other day, and I wanted to share it with anyone who will listen.

## The Scene

I was adding a new activity to a compose multiplatform project, on the Android side.
Not to get too much into the weeds, but, I was adding an activity that would open when a user wants to manage Wi-Fi and
Mobile Data usage.

```xml

<activity
        android:name=".NotificationSettingsActivity"
        android:exported="true">
    <intent-filter>
        <action android:name="android.intent.action.MANAGE_NETWORK_USAGE"/>
        <category android:name="android.intent.category.DEFAULT"/>
    </intent-filter>
</activity>
```

Definitely not needed, but this is for a side project, so, I added it.

I recently switched over this side project to using Nav3, so there is still a lot to learn.
The project is a unique one since I still have the traces of nav2, but I created a wrapper over navigation since there's
a chance this isn't the final navigation we get.
I have an interface for the navigation that allows me to choose which navigation I want to use.

```kotlin
interface NavigationActions {
    fun add(navKey: NavKey)
    fun popBackStack()
    //...
}

class Navigation3Actions(val navBackStack: MutableList<NavKey>) : NavigationActions {
    override fun add(navKey: NavKey) {
        navBackStack.add(navKey)
    }

    override fun popBackStack() {
        navBackStack.removeLast()
    }
}

class Navigation2Actions(
    private val navController: NavHostController,
) : NavigationActions {
    override fun add(navKey: NavKey) {
        navController.navigate(navKey.route)
    }
    override fun popBackStack() {
        navController.popBackStack()
    }
}
```

That's the main idea of how things are set up. There's a lot more, sure, however, this is the gist of it.

## The Issue

The screen that shows up is a screen that can be navigated to from the main application.
The new activity is a fun thing I wanted to add and use the same screen.
And that is where the problem comes in. The back button on the device works fine, but the back button IconButton on the
screen doesn't.

Okay, what do I do? Just go back to nav2? No. Not a chance! I'm committing to nav3!

Here is how my back button currently is set up.

```kotlin
@Composable
fun BackButton() {
    val navActions = LocalNavActions.current
    IconButton(onClick = { navActions.popBackStack() }) { Icon(Icons.AutoMirrored.Filled.ArrowBack, null) }
}
```

I know, I know. I shouldn't be passing the navActions around like this, but the project is a big one, I'm the only one
working on it, and I'm happy with it.

When I press on the back button, I get a crash???
An `IndexOutOfBoundsException`? But why? Then it hit me!
The navigation is still set up. Even with this new screen in its own activity like this, it's still using the
navActions!

Hm...🤔

## The Solution

So I started looking into things. Searching it online wasn't very helpful, no one else has run into this. This is the
first time nav3 is on compose mulitplatform!
In a recent project, I found this:

```kotlin
val back = LocalOnBackPressedDispatcherOwner.current?.onBackPressedDispatcher
```

Aha! This could work!

```kotlin
back?.onBackPressed()
```

Simple, clean, and it works!

![Screenshot 2025-11-04 at 08.39.35.png](Screenshot 2025-11-04 at 08.39.35.png)

Oh...right...Multiplatform...

The `LocalOnBackPressedDispatcherOwner` is an Android only thing.
What now?

This wasn't the end, I looked around and found this:

```Kotlin
LocalNavigationEventDispatcherOwner.current?.navigationEventDispatcher
```

Iiiiinteresting!

What can I do with this?

![Screenshot 2025-11-04 at 08.42.01.png]($PROJECT_DIR$/writerside/images/Screenshot 2025-11-04 at 08.42.01.png)

Ooookay, not...incredibly helpful.

But it can't end there. So, `NavigationEventInput`...What is that?

![Screenshot 2025-11-04 at 08.50.30.png](Screenshot 2025-11-04 at 08.50.30.png)

Okay! Cool...What? I get the gist of it...But now how do I use it?
Is there anything I can already use? I have a motto to never recreate the wheel.
So let's go digging! I need something that will work on all platforms.

![Screenshot 2025-11-04 at 08.53.02.png](Screenshot 2025-11-04 at 08.53.02.png)

Hoo boy, that's a lot. But wait, most of these are platform-specific...Except...one!

`DirectNavigationEventInput`, could you be my savior?

Let's look at this code!

```kotlin
/**
 * An input that can send events to a [NavigationEventDispatcher]. Instead of subclassing
 * [NavigationEventInput], users can create instances of this class and use it directly.
 */
public class DirectNavigationEventInput() : NavigationEventInput() {
    /**
     * Dispatch a back started event with the connected dispatcher.
     *
     * @param event The [NavigationEvent] to dispatch.
     */
    @MainThread
    public fun backStarted(event: NavigationEvent) {
        dispatchOnBackStarted(event)
    }

    /**
     * Dispatch a back progressed event with the connected dispatcher.
     *
     * @param event The [NavigationEvent] to dispatch.
     */
    @MainThread
    public fun backProgressed(event: NavigationEvent) {
        dispatchOnBackProgressed(event)
    }

    /** Dispatch a back cancelled event with the connected dispatcher. */
    @MainThread
    public fun backCancelled() {
        dispatchOnBackCancelled()
    }

    /** Dispatch a back completed event with the connected dispatcher. */
    @MainThread
    public fun backCompleted() {
        dispatchOnBackCompleted()
    }

    /**
     * Dispatch a forward started event with the connected dispatcher.
     *
     * @param event The [NavigationEvent] to dispatch.
     */
    @MainThread
    public fun forwardStarted(event: NavigationEvent) {
        dispatchOnForwardStarted(event)
    }

    /**
     * Dispatch a forward progressed event with the connected dispatcher.
     *
     * @param event The [NavigationEvent] to dispatch.
     */
    @MainThread
    public fun forwardProgressed(event: NavigationEvent) {
        dispatchOnForwardProgressed(event)
    }

    /** Dispatch a forward cancelled event with the connected dispatcher. */
    @MainThread
    public fun forwardCancelled() {
        dispatchOnForwardCancelled()
    }

    /** Dispatch a forward completed event with the connected dispatcher. */
    @MainThread
    public fun forwardCompleted() {
        dispatchOnForwardCompleted()
    }
}
```

...Uh huh...Well! In for a penny, in for a pound!

Let's try it out!

```kotlin
@Composable
fun BackButton() {
    val navEvent = LocalNavigationEventDispatcherOwner.current?.navigationEventDispatcher

    val navInput = remember { DirectNavigationEventInput() }

    DisposableEffect(Unit) {
        navEvent?.addInput(navInput)
        onDispose { navEvent?.removeInput(navInput) }
    }

    IconButton(
        onClick = { navInput.backCompleted() }
    ) { Icon(Icons.AutoMirrored.Filled.ArrowBack, null) }
}
```

And it works! And it works perfectly! On the single screen AND in the main application with full navigation!

## Conclusion

This was a very interesting issue to figure out.
With Compose Multiplatform still being pretty young and Nav3 a baby, there's still a ton to learn.

Nav3 is still in beta, so I'm expecting there to be changes and improvements.
For example, there's still no deep linking! 😁 But I'm sure it'll get there.

I'm excited to see what the future holds for Compose Multiplatform and Nav3!

I hope this helps someone else out there!
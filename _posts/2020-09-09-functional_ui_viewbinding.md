---
layout: post
title: "Functional UI with ViewBinding"
tags: [android,viewbinding,ui]
comments: true
---

Recently, I have to work on a feature with 2 different looking UIs for different configurations, but otherwise have the exact same behaviour. This meant using different view types and setting them up differently, but they end up rendering the same state and calling the same callbacks.

This got me thinking on how to structure UI code such that it is flexible and easy to replace/refactor as a single component, and model it to be a function of a state.

``` kotlin
// being able to do this would be great!

if (someCondition) {
    renderALayout(state)
} else {
    renderBLayout(state)
}
```

Until Jetpack Compose comes along and saves our lives, let's see how it can be done with what we have now.

## Separating UI from Activity/Fragment

Because UI (layouts and views) are always inflated in activities/fragments, it is extremely common and understandable to treat them as the *view* layer (the V in MVx architecture patterns), and write UI code in these classes.

However, activities and fragments in reality are more like controllers than views. It is easier to think of them as a host for our UIs rather than the UI itself.

One way to seperate the UI code from activities/fragments is to write the UI code as an extension function of the generated binding of our layout.

``` kotlin
// Assuming we are designing a user profile screen,
// and have a `user_profile_layout.xml` file.

fun UserProfileLayoutBinding.render(userProfile: UserProfile) {
    usernameTextView.text = userProfile.username
}
```

``` kotlin
// UserProfileFragment.kt

class UserProfileFragment : Fragment(R.layout.user_profile_layout) {

    private val viewModel by viewModels<UserProfileViewModel>()
    
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        val binding = UserProfileLayoutBinding.bind(view)

        viewModel.userProfileLiveData.observe(viewLifecycleOwner, Observer { userProfile ->
            binding.render(userProfile)
        })
    }
}
```

We now have all our UI code in a function!

This keeps the fragment simple, which just connects the UI and the ViewModel together. There is no need to worry about clearing view references or [nulling out the binding](https://developer.android.com/topic/libraries/view-binding#fragments).

When there is a redesign, or when the complexity of the UserProfile object increases with more data to render, the code in the fragment does not need to change. We only have to update the `render` extension function!

## Initial setup and callbacks

The `render` function now works only for extremely simple UI.

In order to handle user interactions, we will have to pass in some callbacks. It is also likely that there is some initial setup code for the UI, such as setting up the Adapter/LayoutManager/ItemDecorations for a RecyclerView.

``` kotlin
// with a more complicated `user_profile_layout.xml`

fun UserProfileLayoutBinding.render(
    onFollow: () -> Unit,
    onPostClick: (Post) -> Unit,
    userProfile: UserProfile
) {
    usernameTextView.text = userProfile.username
    
    followButton.setOnClickListener { onFollow() }
    
    val postsAdapter = PostsAdater(onPostClick)
    postsRecyclerView.adapter = postsAdapter
    postsRecyclerView.layoutManager = LinearLayoutManager(root.context)
    postsAdapter.submitList(userProfile.posts)
}
```

``` kotlin
// UserProfileFragment.kt

class UserProfileFragment : Fragment(R.layout.user_profile_layout) {

    private val viewModel by viewModels<UserProfileViewModel>()
    
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        val binding = UserProfileLayoutBinding.bind(view)

        viewModel.userProfileLiveData.observe(viewLifecycleOwner, Observer { userProfile ->
            binding.render(
                onFollow = viewModel::onFollow,
                onPostClick = { post -> 
                    // navigate to separate screen
                },
                userProfile = userProfile
            )
        })
    }
}
```

Writing the `render` function like this is definitely not ideal, since we are doing the setup every time the LiveData emits a new `UserProfile` and we want to re-render the UI.

The solution will be to split the setup code and the rendering code. This will require a change in the function to return `(UserProfile) -> Unit` instead of `Unit`.

``` kotlin
fun UserProfileLayoutBinding.renderer(
    onFollow: () -> Unit,
    onPostClick: (Post) -> Unit
) : (UserProfile) -> Unit {
    
    // setup code 

    followButton.setOnClickListener { onFollow() }

    val postsAdapter = PostsAdater(onPostClick)
    postsRecyclerView.adapter = postsAdapter
    postsRecyclerView.layoutManager = LinearLayoutManager(root.context)

    // rendering code

    return { userProfile ->
        usernameTextView.text = userProfile.username
        postsAdapter.submitList(userProfile.posts)
    }
}
```

``` kotlin
// UserProfileFragment.kt

class UserProfileFragment : Fragment(R.layout.user_profile_layout) {

    private val viewModel by viewModels<UserProfileViewModel>()
    
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        val binding = UserProfileLayoutBinding.bind(view)
        val renderer = binding.renderer(
            onFollow = viewModel::onFollow,
            onPostClick = { post -> 
                // navigate to separate screen
            }
        )

        viewModel.userProfileLiveData.observe(viewLifecycleOwner, Observer { userProfile ->
            renderer(userProfile)
        })
    }
}
```

## Kotlin functional interface
With [functional interfaces](https://kotlinlang.org/docs/reference/fun-interfaces.html) introduced in Kotlin 1.4, the return type can be made more concise.

``` kotlin
fun interface Renderer<T> {
    fun render(t: T)
}
```

``` kotlin
fun UserProfileLayoutBinding.renderer(
    onFollow: () -> Unit,
    onPostClick: (Post) -> Unit
) : Renderer<UserProfile> {
    
    // setup code 

    followButton.setOnClickListener { onFollow() }

    val postsAdapter = PostsAdater(onPostClick)
    postsRecyclerView.adapter = postsAdapter
    postsRecyclerView.layoutManager = LinearLayoutManager(root.context)

    // rendering code

    return Renderer { userProfile ->
        usernameTextView.text = userProfile.username
        postsAdapter.submitList(userProfile.posts)
    }
}
```

## Just a reference to a function

Back to my problem.

Using this approach, I've written two separate layout with very different UI, but gracefully handling it in the Fragment without having the Fragment look like a mess.

``` kotlin
fun LayoutABinding.renderer(
    callbacks: () -> Unit
    // ...
) : Renderer<MyState> {
    // setup for layout A

    return Renderer {
        // rendering code
    }
}

fun LayoutBBinding.renderer(
    callbacks: () -> Unit
    // ...
) : Renderer<MyState> {
    // setup for layout B

    return Renderer {
        // rendering code
    }
}
```

``` kotlin
// MyFragment.kt

class MyFragment : Fragment() {

    fun onCreateView(inflater: LayoutInflater, container: ViewGroup, savedInstanceState: Bundle?): View? {
        val layout = if (conditionForLayoutA) {
            R.layout.layout_a
        } else {
            R.layout.layout_b
        }
        return inflater.inflate(layout, container, false)
    }

    fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        val renderer = if (conditionForLayoutA) {
            LayoutABinding.bind(view).renderer(
                // callbacks ...
            )
        } else {
            LayoutBBinding.bind(view).renderer(
                // callbacks ...
            )
        }

        viewModel.liveData.observe(viewLifecycleOwner, Observer { myState ->
            renderer.render(myState)
        })
    }
}
```

My fragment doesn't have to know what my UI looks like. It just observes the ViewModel for the state and pass it to the rendering function.

Solves my problem nicely!

## Conclusion

By writing the UI code separately as a function, we can have all UI related code and variables grouped together instead of being placed across the activity/fragment. All the activity/fragment needs is a reference to the UI rendering function, and acts as the coordinator between the ViewModel and the UI.

This allows any refactoring/redesign of UI without affecting the activity/fragment. The activity/fragment can be free of UI-related code, and only doing work such as navigation/intent handling.

It becomes easy to swap out one UI implementation for another without any big changes to the activity/fragment. This makes it way too easy to do A/B testing of different layouts!

It also plays nicely with unidirectional data flow!

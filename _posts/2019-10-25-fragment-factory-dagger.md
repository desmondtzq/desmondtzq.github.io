---
layout: post
title: "FragmentFactory with Dagger"
tags: [android,fragment,dagger,dependency injection]
comments: true
---

Injecting dependencies is most useful when done through the constructor, but due to the limitiations of the Android framework, we are not able to do this for framework classes such as Activities and Fragments. This is because they can be recreated by the system, and the system will always use the empty, no-argument constructor to create instances of these classes.

However, with the introduction of FragmentFactory, it is now possible to inject dependencies through the constructor of a Fragment!

Introduced in [androidx.fragment 1.1.0](https://developer.android.com/jetpack/androidx/releases/fragment#version_110_3):

> You can now set a FragmentFactory on a FragmentManager to manage the creation of fragment instances, removing the strict requirement to have a no-argument constructor.

## TLDR of this post:
- FragmentFactory -> constructor injection of Fragments
- FragmentFactory with Dagger -> eliminate use of patterns such as Service Locator, manual DI, dagger-android

## Setting up the FragmentFactory

Using [Dagger multibindings](https://dagger.dev/multibindings) (similar to the multiinding approach of ViewModelFactory in the [GithubBrowserSample](https://github.com/android/architecture-components-samples/tree/master/GithubBrowserSample)), we are able to have a FragmentFactory which knows how to create our Fragments with `@Inject` annotated constructors.

``` kotlin
class DaggerFragmentFactory @Inject constructor(
    private val providers: Map<Class<out Fragment>, @JvmSuppressWildcards Provider<Fragment>> 
) : FragmentFactory {
    
    override fun instantiate(classLoader: ClassLoader, className: String) {
        val fragmentClass = loadFragmentClass(classLoader, className)
        
        // if the provider for fragmentClass is not in the providers map, 
        // it will default to the the no-args constructor of the Fragment, 
        // or crash if the constructor is not available
        return providers[fragmentClass]?.get() ?: super.instantiate(classLoader, className)
    }
}
```

We then have to bind our fragment so that dagger knows how to provide it to the `providers` Map in DaggerFragmentFactory.

``` kotlin
// MyFragment.kt
// we can now inject dependencies through the constructor

class MyFragment @Inject constructor(
    // dependencies,
    // more dependencies
) : Fragment() {
    // ...
}

// FragmentsModule.kt
// this is required for dagger to know how to inject our fragments into the Map in FragmentFactory

@Module
abstract class FragmentsModule {
    @Binds
    @IntoMap
    @FragmentKey(MyFragment::class)
    abstract fun myFragment(fragment: MyFragment): Fragment
}

// this is the key to associate the binded value
// check out https://dagger.dev/api/latest/dagger/MapKey.html

@MustBeDocumented
@Target(
AnnotationTarget.FUNCTION,
AnnotationTarget.PROPERTY_GETTER,
AnnotationTarget.PROPERTY_SETTER
)
@Retention(AnnotationRetention.RUNTIME)
@MapKey
annotation class FragmentKey(val value: KClass<out Fragment>)

```

And then finally, set the factory in the host activity of the fragments. It is important to do it before `super.onCreate`, because the activity may have to  recreate fragments during `onCreate`, and it will need the custom FragmentFactory to do so.

``` kotlin
class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        val factory = (application as MyApp).appComponent.fragmentFactory()
        supportFragmentManager.fragmentFactory = factory
        super.onCreate(savedInstanceState)

        // the rest of the code
    }
}
```

With this all set up, we can now perform constructor injection for Fragments, as things should have been!

## Injecting ViewModels

Very often, the dependency injected into a Fragment is a ViewModelProvider.Factory, so that the Fragment is able to get an instance of the required ViewModel using the injected factory. An example of this can be seen, again from the GithubBrowserSample.

Now, the question: can we inject `ViewModel` into the Fragment through the constructor?

Yes, absolutely! Let's find out how.

We just have to make sure that the ViewModel can be created by Dagger, by adding the `@Inject` annotation to its constructor. It shouldn't have a scope annotation, because the scope of the ViewModel depends on the lifecycle of the Fragment/Activity.

``` kotlin
class MyViewModel @Inject constructor(
    private val repo: Repository
) : ViewModel() {
    // ...
}

```

...And include it in the Fragment's constructor!

``` kotlin
class MyFragment @Inject constructor(
    private val provider: Provider<MyViewModel>
) : Fragment() {
    // ...
}
```

Notice that we are injecting `Provider<MyViewModel>`, and not `MyViewModel` directly. This is because the ViewModel should be created only when necessary, and with a `Provider`, the ViewModel is not actually created. The `Provider` acts as a factory of the ViewModel, and the ViewModel will only be created when `provider.get()` is invoked.

### Using `Provider<MyViewModel>` for ViewModelProvider.Factory

With the injected `Provider<MyViewModel>`, we can create a ViewModelProvider.Factory from the provider, and use it to retrieve an instance of MyViewModel.

``` kotlin
class MyFragment @Inject constructor(
    private val provider: Provider<MyViewModel>
) : Fragment() {

    private val viewModel: MyViewModel by viewModels {
        object : ViewModelProvider.Factory {
            override fun <T : ViewModel> create(modelClass: Class<T>): T {
                @Suppress("UNCHECKED_CAST")
                return provider.get() as T
            }
        }
    }
}

```

Doing this for every ViewModel is probably too much code to write, so we can create an extension function to reduce the boilerplate.

``` kotlin
inline fun <reified T : ViewModel> Fragment.viewModels(
    noinline ownerProducer: () -> ViewModelStoreOwner = { this },
    defaultArgs: Bundle? = null,
    crossinline provider: (SavedStateHandle) -> T
) = viewModels<T>(ownerProducer) {

    // using AbstractSavedStateViewModelFactory by default,
    // so we can have access to a SavedStateHandle 
    // if ever we need it in our ViewModels

    object : AbstractSavedStateViewModelFactory(this, defaultArgs) {
        override fun <T : ViewModel> create(
            key: String,
            modelClass: Class<T>,
            handle: SavedStateHandle
        ): T {
            @Suppress("UNCHECKED_CAST")
            return provider(handle) as T
        }
    }
}
```

Using the extension function, we only need to pass in a lambda for creating the ViewModel.
``` kotlin
class MyFragment @Inject constructor(
    private val provider: Provider<MyViewModel>
) : Fragment() {

    private val viewModel: MyViewModel by viewModels { provider.get() }
}
```


### Runtime dependencies

But what if I have runtime dependencies that I want to pass to the ViewModel constructor, such as an item's id from the Fragment arguments?

Well, not to worry. This is where we can tools such as [AssistedInject](https://github.com/square/AssistedInject), which can help to create generated Factory of the AssistedInject class. We then inject the Provider of the generated Factory instead of the ViewModel, while still leveraging Dagger to do most of the dependency injection work behind the scenes.

``` kotlin
class MyViewModel @AssistedInject constructor(
    @Assisted private val id: String,
    private val repo: Repository
) {
    @AssistedInject.Factory 
    interface Factory {
        fun create(id: String): MyViewModel
    }
    // ...
}

class MyFragment @Inject constructor(
    private val provider: Provider<MyViewModel.Factory>
) : Fragment() {

    private val viewModel: MyViewModel by viewModels {
        val id = arguments.getString("arg_id")!!
        provider.get().create(id)
    }
}

```

Jake Wharton gave an excellent talk [Helping Dagger Help You](https://jakewharton.com/helping-dagger-help-you/) which explains everything to you need to learn about assisted injection.

## Navigating between fragments

If you are not using the AndroidX Navigation library, and want to perform fragment transaction manually, you can use the add/replace method that takes the Fragment's Class instead of the fragment instance.

Instead of this
``` kotlin
val fragment = MyFragment.newInstance(id)
parentFragmentManager.commitNow {
    replace(R.id.container, fragment)
}
```

You can do this

``` kotlin
val args = Bundle().apply { 
    putString("arg_id", id)
}
parentFragmentManager.commitNow {
    replace(R.id.container, MyFragment::class.java, args)
}

// alternatively, using fragment-ktx
parentFragmentManager.commitNow {
    replace<MyFragment>(R.id.container, args = args)
}
```

> The add/replace overloads that takes a Class<? extends Fragment> and optional Bundle of arguments is only added in [androidx.fragment 1.2.0-alpha02](https://developer.android.com/jetpack/androidx/releases/fragment#1.2.0-alpha02).

## Conclusion

With the introduction of FragmentFactory, we can now finally use constructor injection everywhere in our app! (Well, almost... we still cannot do that for activities yet, unless your min sdk is 28)

Bonus points for using a single activity architecture, because now we only need to the access AppComponent from the Activity to get a custom FragmentFactory and set it to the fragmentManager, and use constructor injection everywhere else! For multi-activities setup, we can do the same for each activity, or have a BaseActivity that does this that every Activity can extends from.
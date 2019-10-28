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

## FragmentFactory

The FragmentFactory class, as the name suggests, is the factory for fragments, and can be assigned to a FragmentManager for it to instantiate new instances of fragments. 

The FragmentFactory has a single method `instantiate(ClassLoader, String)`, which takes in the context's ClassLoader and the class name of the fragment. The default implementation of the `instantiate` method simply attempts to invoke the empty, no-arg constructor of the Fragment, or throws an error if the empty constructor is not available.

## Setting up a custom FragmentFactory with Dagger

With Dagger, we will be able to create a custom FragmentFactory that knows how to create any Fragment that we have.

### Annotate Fragment's constructor

To start off, we need to let Dagger know how to create our Fragments. This can be achieved by just adding the `@Inject` annotation to the Fragment's constructor.

``` kotlin
class MyFragment @Inject constructor() : Fragment()
```

You're able to add any dependencies you need for the Fragment in the constructor, as long as the dependency is made available to the Dagger graph at compile time.

### Custom FragmentFactory

Now that Dagger knows how to create our fragments, we're able to create a custom FragmentFactory that knows how to create all our fragments using [Map Multibindings](https://dagger.dev/multibindings). This approach is similar to the multiinding approach of the [ViewModelFactory](https://github.com/android/architecture-components-samples/blob/master/GithubBrowserSample/app/src/main/java/com/android/example/github/viewmodel/GithubViewModelFactory.kt) in [GithubBrowserSample](https://github.com/android/architecture-components-samples/tree/master/GithubBrowserSample).

This is basically injecting to our FragmentFactory a map of `Class<out Fragment>` key with a `Provider<Fragment>` value , and using this map to instantiate fragments.

A `Provider<T>` is basically an object which is able to provide an instance of `T` through the `get` method. Any class that is part of the dagger graph can be returned as `Provider<T>` instead of just `T`.

By associating `Provider<T>` with a key of `Class<T>`, it means that given a `Class<T>`, we are able to get `Provider<T>`, and with the provider, return an instance of `T`.

``` kotlin
class DaggerFragmentFactory @Inject constructor(
    private val providers: Map<Class<out Fragment>, @JvmSuppressWildcards Provider<Fragment>> 
) : FragmentFactory {
    
    override fun instantiate(classLoader: ClassLoader, className: String): Fragment {
        // loadFragmentClass is a static method of FragmentFactory
        // and will return the Class of the fragment
        val fragmentClass = loadFragmentClass(classLoader, className)

        // we will then be able to use fragmentClass to get the provider
        // of the Fragment from the providers map
        val provider = providers[fragmentClass]

        if (provider != null) {
            return provider.get()
        }

        // The provider for the fragment could be null
        // if the Fragment class is not binded to the Daggers graph
        // in this case, we will default to the default implementation
        // which will attempt to instantiate the Fragment 
        // through the no-arg constructor
        return super.instantiate(classLoader, className)
    }
}
```

### Binding Fragment to the Dagger graph

The last step in the setup is to populate the map of Class<T> to Provider<T> in DaggerFragmentFactory.

We will first have to define a [MapKey](https://dagger.dev/api/latest/dagger/MapKey.html), in order to specify a class type as a key of the map. We can name it `FragmentKey`, and it will take in the KClass of a Fragment.

``` kotlin
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

Now, we can define entries for the map by binding our Fragment classes into the map using the `FragmentKey` annotation we defined previously.

The key of the map entry will be the KClass of the Fragment, and the value will be the `Provider` of the Fragment defined in the parameter of the binding function.

``` kotlin
@Module
abstract class FragmentsModule {

    // entry in map: Class<MyFragment> to Provider<MyFragment>
    @Binds
    @IntoMap
    @FragmentKey(MyFragment::class)
    abstract fun myFragment(fragment: MyFragment): Fragment

    // entry in map: Class<MyOtherFragment> to Provider<MyOtherFragment>
    @Binds
    @IntoMap
    @FragmentKey(MyOtherFragment::class)
    abstract fun myOtherFragment(fragment: MyOtherFragment): Fragment

    // ...
}
```

And we are all done for the Dagger part! We can now perform constructor injection for Fragments, as things should have been!

## Setting a custom FragmentFactory

In order to start using a custom FragmentFactory, we will have to set it to the FragmentManager of the host Activity.

It is important to do it before `super.onCreate`, because the activity may have to  recreate fragments during `onCreate`, and it will need the custom FragmentFactory to do so.

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

The FragmentManager will delegate the instantiating of Fragments to the FragmentFactory we have defined. This means that we should not be instantiating instances of Fragments manually using the `newInstance` approach, and should rely on the FragmentManager to instantiate fragments for us, either manually or using the AndroidX Navigation library (2.1.0-alpha01 and above).

## Navigating between fragments

If you are not using the AndroidX Navigation library, and want to perform fragment transaction manually, you can use the add/replace method that takes the Fragment's Class instead of the fragment instance.

Instead of using the `newInstance` approach to manually instantiate a Fragment,
``` kotlin
val fragment = MyFragment.newInstance(id)
parentFragmentManager.commitNow {
    replace(R.id.container, fragment)
}
```

we can use the Fragment's Class for the transaction instead.

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

## Injecting ViewModels

Very often, the dependency injected into a Fragment is a ViewModelProvider.Factory, so that the Fragment is able to get an instance of the required ViewModel using the injected factory. An example of this can be seen, again from the [GithubBrowserSample](https://github.com/android/architecture-components-samples/tree/master/GithubBrowserSample).

Let's find out how we can make use of constructor injection of a Fragment to provide a ViewModel to the Fragment.

### ViewModel with Dagger

First of all, we have to make sure that the ViewModel can be created by Dagger, by adding the `@Inject` annotation to its constructor. It shouldn't have a scope annotation, because the scope of the ViewModel depends on the lifecycle of the Fragment/Activity.

``` kotlin
class MyViewModel @Inject constructor(
    private val repo: Repository
) : ViewModel() {
    // ...
}
```

...And include a `Provider` of the ViewModel in the Fragment's constructor!

``` kotlin
class MyFragment @Inject constructor(
    private val provider: Provider<MyViewModel>
) : Fragment() {
    // ...
}
```

Notice that we are injecting `Provider<MyViewModel>`, and not `MyViewModel` directly. 

The ViewModel should be instantiated only when necessary, when the ViewModel has not been instantiated yet, i.e., the ViewModel is not available in the Fragment's ViewModelStore.

By injecting a `Provider` of the ViewModel, the ViewModel is not actually instantiated. The `Provider` acts as a factory, and will only instantiate the ViewModel when `provider.get()` is invoked.

### Using `Provider<MyViewModel>` for ViewModelProvider.Factory

With the injected `Provider<MyViewModel>`, we can create a ViewModelProvider.Factory from the provider, and use it to retrieve an instance of MyViewModel.

``` kotlin
class MyFragment @Inject constructor(
    private val provider: Provider<MyViewModel>
) : Fragment() {

    // create a ViewModelProvider.Factory with Provider<MyViewModel>

    private val viewModelFactory =
        object : ViewModelProvider.Factory {
            override fun <T : ViewModel> create(modelClass: Class<T>): T {
                @Suppress("UNCHECKED_CAST")
                return provider.get() as T
            }
        }

    // by viewModels is the delegate provided in fragment-ktx

    private val viewModel: MyViewModel by viewModels { viewModelFactory }
}

```

Doing this for every ViewModel in each Fragment is probably too much code to write, so we can create an extension function to reduce the boilerplate.

``` kotlin
/**
 * The ViewModelStoreOwner controls the scope of the ViewModel.
 * It may be overridden with a different ViewModelStoreOwner, 
 * such as the host Activity or the parent fragment, in order to
 * scope the lifetime of the ViewModel to the lifetime of the 
 * ViewModelStoreOwner that is passed in.
 */
inline fun <reified T : ViewModel> Fragment.viewModelWithProvider(
    noinline ownerProducer: () -> ViewModelStoreOwner = { this },
    crossinline provider: () -> T
) = viewModels<T>(ownerProducer) {
    object: ViewModelProvider.Factory {
        override fun <T : ViewModel?> create(modelClass: Class<T>): T {
            @Suppress("UNCHECKED_CAST")
            return provider.invoke() as T
        }
    }
}
```

Alternatively, if you are using [SavedStateHandle](https://developer.android.com/jetpack/androidx/releases/lifecycle#viewmodel-savedstate_version_100_2)

``` kotlin
inline fun <reified T : ViewModel> Fragment.savedStateViewModelWithProvider(
    noinline ownerProducer: () -> ViewModelStoreOwner = { this },
    noinline savedStateRegistryOwnerProducer: () -> SavedStateRegistryOwner = { this },
    defaultArgs: Bundle? = null,
    crossinline provider: (SavedStateHandle) -> T
) = viewModels<T>(ownerProducer) {

    object : 
        AbstractSavedStateViewModelFactory(savedStateRegistryOwnerProducer(), defaultArgs) {
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

Using the extension function we have defined, we only need to pass in a lambda for instantiating the ViewModel, and not have to create a ViewModelProvider.Factory manually.

``` kotlin
class MyFragment @Inject constructor(
    private val provider: Provider<MyViewModel>
) : Fragment() {

    private val viewModel: MyViewModel by viewModelWithProvider { provider.get() }
}
```


### Runtime dependencies

But what if I have runtime dependencies that I want to pass to the ViewModel constructor, such as an item's id from the Fragment arguments, or a SavedStateHandle?

Well, not to worry. This is where [AssistedInject](https://github.com/square/AssistedInject) comes in handy. With AssistedInject, it can help to generate Factory of the AssistedInject class, helping us use dagger for injecting dependencies known at compile-time, while allowing us to pass in dynamic arguments at run-time.

Let's say we have a ViewModel with a constructor that takes in a String id, and a repository. The id is only known at runtime, which is to be retrieved from the Fragment's argument, while the repository is known at compile-time.

``` kotlin
class MyViewModel @AssistedInject constructor(
    @Assisted private val id: String,
    private val repo: Repository
) {
    // define a factory interface with a single method with a signature
    // which takes in the runtime argument, and returns the class to be constructed
    @AssistedInject.Factory 
    interface Factory {
        fun create(id: String): MyViewModel
    }
}
```

 We then inject the Provider of the Factory of the ViewModel, instead of the Provider of the ViewModel.

``` kotlin
class MyFragment @Inject constructor(
    private val provider: Provider<MyViewModel.Factory>
) : Fragment() {

    private val viewModel: MyViewModel by viewModelWithProvider {
        val id = arguments.getString("arg_id")!!
        provider.get().create(id)
    }
}
```

Jake Wharton gave an excellent talk [Helping Dagger Help You](https://jakewharton.com/helping-dagger-help-you/) which explains everything to you need to learn about assisted injection.

## Conclusion

With the introduction of FragmentFactory, we can now finally use constructor injection everywhere in our app! (Well, almost... we still cannot do that for activities yet, unless your min sdk is 28)

Bonus points for using a single activity architecture, because now we only need to the access AppComponent from the Activity to get a custom FragmentFactory and set it to the fragmentManager, and use constructor injection everywhere else! For multi-activities setup, we can do the same for each activity, or have a BaseActivity that does this that every Activity can extends from.
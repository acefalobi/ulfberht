# Ulfberht

[ ![Bintray](https://api.bintray.com/packages/drummer-aidan/maven/ulfberht/images/download.svg) ](https://bintray.com/drummer-aidan/maven/ulfberht/_latestVersion)
[![Build Status](https://travis-ci.org/afollestad/ulfberht.svg)](https://travis-ci.org/afollestad/ulfberht)
[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)

> The Vikings were among the fiercest warriors of all time. Yet only a select few carried the 
> ultimate weapon of their era: the feared Ulfberht sword. Fashioned using a process that would 
> remain unknown to the Vikings' rivals for centuries, the Ulfberht was a revolutionary high-tech 
> tool as well as a work of art.

*A little more bad-ass than a Dagger, huh?*

Dependency injection is a technique in which an application supplies dependencies of an object. 
"Dependencies" in this context are not dependencies like in a Gradle file. Dependencies are services 
(i.e. APIs, classes) that are needed in certain parts of your code.

Dependency injection enables you to pass around services without manual construction - it keeps 
track of everything for you and injects things where they are needed. The need for this is more 
apparent when you deal with very large applications.

---

# Table of Contents

1. [Why Choose Ulfberht?](#why-choose-ulfberht)
2. [Gradle Dependency](#gradle-dependency)
3. [Modules](#modules)
    1. [Binding](#binding)
    2. [Providing](#providing)
    3. [Singletons](#singletons)
    4. [Qualifiers](#qualifiers)
4. [Components](#components)
    1. [Basics](#basics)
    2. [Parenting](#parenting)
    4. [Scoping](#scoping)
5. [Putting Modules and Components to Use - Injection](#putting-modules-and-components-to-use---injection)
6. [Runtime Dependencies](#runtime-dependencies)
7. [Android](#android)
    1. [ScopeOwners](#scope-owners)
    2. [ViewModels](#viewmodels)

---
 
# Why Choose Ulfberht?

I wrote Ulfberht as an experiment to see if I could make [Dagger](https://github.com/google/dagger) 
style dependency injection a bit easier, and quicker to pickup for newbies. I wanted something 
lightweight, with less Boilerplate code and more automation. I wanted something annotation-processor 
based rather than reflection-based, while still being written in Kotlin. And I wanted better built-in 
scoping support, especially on Android. _This is the result._

You may be wondering what makes this library different than KOIN or other Kotlin "DI" libraries? 
Ulfberht uses annotation processing to do actual dependency injection vs. plain service location. 
A big example of this difference is that you don't need to manually tell this library how to fill 
constructor parameters, they are filled for you by generated code.

---
 
# Gradle Dependency

```gradle
dependencies {
  implementation "com.afollestad:ulfberht:0.4.0"
  kapt "com.afollestad:ulfbert-processor:0.4.0"
}
```

---

# Modules

A module is a class that gives instructions for dependency instantiation.

### Binding

One way a module can instruct object instantiation is through binding. If you've used Dagger, this 
should be straightforward. 

In this example below, we bind an interface with a concrete implementation. Whenever you inject 
`Demo`, you inject the `DemoImpl` implementation of it.

Taking this interface and implementation...

```kotlin
interface Demo {
  fun myMethod()
}

class DemoImpl : Demo {
  override fun myMethod() {
    ...
  }
}
```

...they can be bound in a `@Module` interface:

```kotlin
@Module
interface DemoModule {
  @Binds 
  fun demoClass(impl: DemoImpl): Demo
}
```

---

If `DemoImpl` itself had dependencies in its constructor, those must be bound or provided as well 
so that they can be injected too...

```kotlin
interface SomethingElse

class SomethingElseImpl : SomethingElse

interface Demo {
  fun myMethod()
}

class DemoImpl(
  val somethingElse: SomethingElse
) : Demo {
  override fun myMethod() {
    ...
  }
}
```

...with a module setup like this:

```kotlin
@Module
interface DemoModule {
  @Binds
  fun demoClass(impl: DemoImpl): Demo
  
  @Binds
  fun somethingElse(impl: SomethingElseImpl): SomethingElse
}
```

### Providing

Another way a module can instruct object instantiation is through providing. This should also be a 
familiar concept for Dagger users. 

Providing is more flexible than binding. You tell the library how instantiation should happen, and 
what you provide _does not_ need all of its constructor parameters to be injectable. Notice that 
a module which can use `@Provides` must be a `abstract class` rather than an `interface`.

```kotlin
class SomethingElse
class Demo(
  val somethingElse: SomethingElse
) {
  fun myMethod() {
    ...
  }
}

@Module
abstract class DemoModule {
  @Provides 
  fun demoClass(): Demo {
    val somethingElse = SomethingElse()
    return Demo(somethingElse)
  }
}
```

`@Provides` methods can have parameters, which the library will fill from the dependency graph 
as well. This could be a dependency provided in another module or even another component's module.

```kotlin
class SomethingElse
class Demo(
  val somethingElse: SomethingElse
)
 
@Module
interface DemoModule {
  @Provides 
  fun demoClass(
    somethingElse: SomethingElse
  ): Demo {
    return Demo(somethingElse)
  }
  
  @Provides 
  fun somethingElse(): SomethingElse {
    return SomethingElse()
  }
}
```

### Singletons

There's a `@Singleton` annotation that can be used to mark `@Binds` and `@Provides` methods. When 
it's used, a module in a specific component will hold the same instance of the provided object until 
the module is destroyed. *If you were to use the same module in two different components, 
`ComponentA` and `ComponentB`, each component would have a separate singleton instance of what 
you're providing. They wouldn't share between each other.*

```kotlin
@Module
interface DemoModule1 {
  @Binds @Singleton 
  fun demoClass1(impl: Demo1Impl): Demo1
}

@Module
abstract class DemoModule2 {
  @Provides @Singleton 
  fun demoClass2(): Demo2 {
    return Demo2Impl()
  }
}
```

Every time injection pulls `Demo1` and `Demo2`, it will be the same cached instances.

---

# Components

A component is a class that takes a set of [modules](#modules), and knows how to inject what they 
collectively bind/provide into a target object.

### Basics

A basic component looks like this:

```kotlin
@Component(modules = [DemoModule::class])
interface DemoComponent {
  fun inject(target: SomeClass)
}
```

You could include multiple items in the array of the `@Component`'s `modules` parameter. 
You can also define a void (no return type) `inject` method for every class that the component 
can inject into.

### Parenting

A component can have a parent. A component's parent can also have a parent. As you chain 
components together with parenting, you build a graph of object dependencies.

<img src="https://github.com/afollestad/ulfberht/blob/master/art/component_diagram.png?raw=true" />

When you inject something at the bottom of the chain, you're able to inject things that are bound 
or provided throughout the chain all the way up to the top.

It's as simple as adding a parameter to your `@Component` annotations.

```kotlin
@Component(modules = [Module1::class, Module2::class])
interface Component1

@Component(
  parent = Component1::class, 
  modules = [Module3::class, Module4::class]
)
interface Component2

@Component(
  parent = Component1::class,
  modules = [Module5::class, Module6::class]
)
interface Component3

@Component(
  parent = Component2::class,
  modules = [Module7::class, Module8::class]
)
interface Component4 {
  fun inject(target: SomeClass)
}

@Component(
  parent = Component3::class,
  modules = [Module9::class, Module10::class]
)
interface Component5  
```

### Scoping

In many applications, especially mobile applications, you generally do not want to keep things around 
for the entire lifecycle of the app. 

Say you're on a login screen, and need to inject an authentication dependency -- you probably only 
need access to that object on the login screen. Once you leave, that authenticator should go away. 
Scoping allows you to achieve this easily.

---

`Scope`'s are associated with components. When a scope is exited, components in that scope destroy 
themselves along with their modules, and anything that the modules may be storing. To scope a component, 
there's a simple parameter to add on the `@Component` annotation.

```kotlin
// Using constants is encouraged so you can share values throughout your app.
const val FIRST_SCOPE = "first scope"

@Component(
  scope = FIRST_SCOPE, 
  modules = [MyModule::class]
)
interface MyComponent
```

You can then retrieve an instance of this scope. There are a few things you can do with it:

```kotlin
val scope: Scope = getScope(FIRST_SCOPE)

// Hook into its lifecycle, which currently is just an exit event.
scope.addObserver(object : ScopeObserver {
  override fun onExit() {
    ...
  }
})

// You can tell the scope to exit, destroying its children.
scope.exit()
```

When you use the `component<>()` method to retrieve a component with a scope, that component is 
attached to its scope. There can be multiple components in a scope. 

When you call `exit()` on that scope, every component attached to it is destroyed. All modules in 
those components go with them, along with any stored singletons amd child components. The next time 
you call `component<>()` for a destroyed component, a new instance is created.

---

If you use parenting _and_ scoping together:

```kotlin
const val PARENT_SCOPE = "i'm a parent"

@Component(
  scope = PARENT_SCOPE, 
  modules = [Module1::class]
)
interface ParentComponent

@Component(
  parent = ParentComponent::class,
  modules = [Module2::class]
)
interface ChildComponent

// This would destroy both components
getScope(PARENT_SCOPE).exit()
```

**Parent components will destroy all of their children (components and their modules) as well.** 

<img src="https://github.com/afollestad/ulfberht/blob/master/art/component_destruction_diagram.png?raw=true" />

---

# Putting Modules and Components to Use - Injection

To perform injection, you need to retrieve the component that's able to inject into your target.

```kotlin
class MyActivity : AppCompatActivity() {
  @Inject lateinit var someDependency: NeededClass
  
  override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    
    // Perform injection, `SomeComponent` would have an inject() method defined for `MyActivity`
    component<SomeComponent>().inject(this)
    
    // Use the injected dependency!
    someDependency.helloWorld()
  }
}
```

This code assumes that one of the modules going up the graph from `SomeComponent` can supply 
`NeededClass` with a `@Binds` or `@Provides` method.

---

# Qualifiers

Qualifiers are simple identifiers that associate a type that may be broad, like a string, with a 
very specific bound or provided instance. There are a few places where you can specify a qualifier.

First, the `@Binds` and `@Provides` annotations take a qualifier:

```kotlin
const val QUALIFIER_ONE = "one"
const val QUALIFIER_TWO = "two"

@Module
interface DemoModule1 {
  @Binds(QUALIFIER_ONE)
  fun demoClass1(impl: Demo1Impl): Demo1
  
  @Binds(QUALIFIER_ONE)
  fun demoClass2(impl: Demo2Impl): Demo2
}

@Module
abstract class DemoModule2 {
  @Provides(QUALIFIER_TWO)
  fun demoClass1(): Demo1 {
    return Demo1Impl()
  }
  
  @Provides(QUALIFIER_TWO)
  fun demoClass2(): Demo2 {
    return Demo2Impl()
  }
}
```

Second, constructor parameters take a qualifier via a `@Param` annotation:

```kotlin
class SomeInjectedClass(
  @Param(QUALIFIER_ONE) val someDependency: Demo1,
  @Param(QUALIFIER_TWO) val anotherDependency: Demo1
) {
  ...
}
```

Third, `@Provides` method parameters take a qualifier also via the `@Param` annotation:

```kotlin
@Module
abstract class DemoModule2 {
  @Provides 
  fun demoClass1(
    @Param(QUALIFIER_ONE) neededDependency: Demo2
  ): Demo1 {
    return Demo1Impl()
  }
}
```

Finally, the `@Inject` annotation takes a qualifier as well:

```kotlin
class SomeClass {
  @Inject(QUALIFIER_ONE) 
  lateinit var someDependency: Demo1
  @Inject(QUALIFIER_TWO) 
  lateinit var anotherDependency: Demo1

  init {
    component<SomeComponent>().inject(this)
  }
}
``` 

---

# Runtime Dependencies

Sometimes your app may need to be able to inject something that is defined at runtime, something 
that cannot be constructed in a module. A good example of when this would be necessary is in 
an Android application, like if you needed to inject the Application context.

First, you tag constructor parameters that need to be provided at runtime with the `@Param` 
annotation, which is discussed in [Qualifiers](#qualifiers) above.

```kotlin
const val APP_CONTEXT: String = "app_context"

class StringRetriever(
  @Param(APP_CONTEXT) val appContext: Context
) {
  fun getString(@IdRes res: Int): String {
    return appContext.resources.getString(res)
  }
}
```

At injection time, you pass mapped runtime dependencies into the `component` method. They are available 
for injection until the component is destroyed, or its parents destroy it. 

```kotlin
// Should ideally be the same constant above, rather than being defined twice
const val APP_CONTEXT: String = "app_context"

class LoginActivity : AppCompatActivity() {
  @Inject 
  lateinit var stringRetriever: StringRetriever 
  
  override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    
    component<LoginComponent>(
      APP_CONTEXT to applicationContext 
    ).inject(this)
  }
}
```

Runtime dependencies in a component are made available to all of the component's children too. In an Android application, providing the application context at the `Application` level will make it available to all Activities and Fragments that use child components.

---

# Android

### ScopeOwners

On Android, you can automatically attach scopes to `LifecycleOwner`'s, such as:
* `Fragment` (from `androidx.app`)
* `AppCompatActivity`/`FragmentActivity`
* `androidx.lifecycle.ViewModel`

_(these all implement the `LifecycleOwner` interface)_

---

First, setup your components and modules as you would normally:

```kotlin
object ScopeNames {
  const val LOGIN_SCOPE = "scope_login"
  const val MAIN_SCOPE = "scope_main"
}

@Component(modules = [AppModule::class])
interface AppComponent {
  fun inject(app: Application)
}

@Component(
  scope = ScopeNames.LOGIN_SCOPE,
  parent = AppComponent::class,
  modules = [LoginModule::class]
)
interface LoginComponent {
  fun inject(activity: LoginActivity)
}

@Component(
  scope = ScopeNames.MAIN_SCOPE,
  parent = AppComponent::class,
  modules = [MainModule::class]
)
interface MainComponent {
  fun inject(activity: MainActivity)
}
```

Then, you annotate your `LifecycleOwner`'s with the `ScopeOwner` annotation:

```kotlin
@ScopeOwner(LOGIN_SCOPE)
class LoginActivity : AppCompatActivity() {
  
  override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    component<LoginComponent>().inject(this)
  }
}
```

Since `LoginComponent` is being injected, _and_ because it's marked as being in `LOGIN_SCOPE`, it 
will automatically destroy itself when `LoginActivity` is destroyed. 

---

### ViewModels

On Android, injecting AndroidX `ViewModel`'s is supported. You don't have to do anything special, 
just inject the `ViewModel` as you would inject anything else. 

However, you can only inject a `ViewModel` into an `androidx.fragment.app.Fragment` or 
`androidx.fragment.app.FragmentActivity` (includes `AppCompatActivity` and descendants). Why? 
Internally, Ulfberht's generated code delegates through `ViewModelProviders` which must attach to 
an Activity or Fragment.
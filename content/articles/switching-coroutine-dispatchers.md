+++
author = "Raúl Maza"
title = "Switching Coroutine Dispatchers in Android: The good way."
date = "2023-06-04"
description = "Sample article showcasing basic Markdown syntax and formatting for HTML elements."
tags = [
"kotlin",
"android",
"coroutines",
]
categories = [
"dev",
]
series = ["Themes Guide"]
aliases = ["switching-coroutine-dispatchers"]
+++

#### Switching Coroutine Dispatchers in Android: The good way
`Published Jun 04, 2023`
 

In this article, we are going to discuss who should be responsible for switching *Coroutine Dispatchers*.

Remember that *CoroutineDispatcher* is part of the [coroutines library](https://github.com/Kotlin/kotlinx.coroutines), 
it brings the capability of confining Coroutine execution to a specific thread, dispatching it to a thread pool, or letting it run unconfined.

Coroutines library is already providing some build-in coroutine dispatchers, let's see the usage of the most common ones:

- **Default:** CPU-intensive operations.
- **IO:** I/O Operations (read/write files, network calls...)
- **Main:** Specific Android dispatcher for interacting with the UI.

---



Sometimes there is confusion over where should we switch to a specific dispatcher, to avoid confusion we can
follow this principle: [A Coroutine should be safe to call from the main thread](https://developer.android.com/kotlin/coroutines/coroutines-best-practices#main-safe)

> Suspend functions should be main-safe, meaning they're safe to call from the main thread. If a class is doing long-running blocking operations in a coroutine, it's in charge of moving the execution off the main thread using withContext.

Let's see an example **NOT** following the principle:

Imagine we have a use case that has to do some expensive operations over a list:

````kotlin
class ExpensiveOperationsOverListUseCase {
    operator fun invoke(list: List<String>): List<Int> {
        //Do expensive operations and map to a new list...
    }
}
````

Then we may need to use our use case in a ViewModel, let's say that because we developed *ExpensiveOperationsOverListUseCase*
we know that invoking it could be potentially expensive in terms of CPU, so we decided to launch it within a coroutine:

````kotlin
class ExampleViewModel(
    private val expensiveUseCase: ExpensiveOperationsOverListUseCase,
    private val defaultDispatcher: CoroutineDispatcher = Dispatchers.Default //Injecting the dispatcher within the constructor
) : ViewModel() {

    private fun doSomething(list: List<String>) {
        viewModelScope.launch { // By default, build-in viewModelScope uses the main dispatcher
            val result = withContext(defaultDispatcher) { // withContext allow us to change the coroutine context, in this case we use it to change the CoroutineDispatcher to the default
                expensiveUseCase(list)
            }
            // Update UI with result (We are on the main dispatcher again, so it will be safe to interact with the UI)
        }
    }
}
````

Following this approach, everything will work as expected, but we are delegating the responsibility of changing
the CoroutineDispatcher to the ViewModel, in this case, we know that the use case is doing some expensive CPU work because
we developed it, but it might not be the case, so it's a better option to expose a suspend function in our use case
and decide there which dispatcher to use:

````kotlin
class ExpensiveOperationsOverListUseCase(
    private val defaultDispatcher: CoroutineDispatcher = Dispatchers.Default //Injecting the dispatcher within the constructor
) {
    suspend operator fun invoke(list: List<String>): List<Int> = withContext(defaultDispatcher) { //We know that this function is doing heavy operations, so we decide to expose it as suspend, and we are in charge of changing the Dispatcher. Great!
        //Do expensive operations and map to a new list...
    }
}
````

**Notice that we are injecting the CoroutineDispatcher within the constructor, it's also a good practice that makes things
easier for testing but that will be covered on another article.*

````kotlin
class ExampleViewModel(
    private val expensiveUseCase: ExpensiveOperationsOverListUseCase,
) : ViewModel() {

    private fun doSomething(list: List<String>) {
        viewModelScope.launch {
            val result = expensiveUseCase(list) //Now this is safe to call from the main thread
        }
        // Update UI with result
    }
}
````

Now the ViewModel doesn't care about which dispatcher use for executing the use case, the responsibility of switching
the dispatcher is on the closest point to the expensive operation.


---

If we have a look, some common libraries like Retrofit, Apollo Kotlin or Room are already following this approach.

- From [version 2.6.0 ](https://github.com/square/retrofit/blob/master/CHANGELOG.md#version-260-2019-06-05) Retrofit supports suspend modifier, using it on the Api declaration will execute the network call in a background dispatcher out of the box.

````kotlin
@GET("users/{id}")
suspend fun user(@Path("id") id: Long): User
````
- It's the same if we want to invoke a Graphql query with [Apollo Kotlin](https://github.com/apollographql/apollo-kotlin). You can check the [documentation](https://www.apollographql.com/docs/kotlin/essentials/queries/).
> By default, Apollo Kotlin offloads I/O work to a background thread, which means it's safe to start GraphQL operations on the main thread. The result is also dispatched to the calling thread, and you can use the response directly to update your data.  
> On the JVM, the I/O work is using Dispatchers.IO by default. You can change this dispatcher with ApolloClient.Builder.dispatcher.

---

#### Conclusion
As you see, some popular libraries follow the principle: **A Coroutine should be safe to call from the main thread**
so why not we do the same in our code base? Let's do it!

---
title: Android ViewModel StateMachine
layout: post
parent: Articles
---
# {{ page.title }}

Are you faced with a UI best represented by a state machine, but for some reason every other tutorial you can find is terrible and only tells you how to make a generic FSM (or even just calls a sealed interface representing state a FSM), something you obviously already know? Well I was, and it has only made me hate Medium more. I came up with a fairly workable solution that mostly respects the ViewModel pattern, and I am going to document it here.

My key abstraction is to represent interaction with the ViewModel as a series of actions and events passed to and from the State. This is based on my experience working with complicated Jetpack Compose widgets (future me: insert a link if I ever write an article about that). Here is an example implementation of the base state machine:

```kotlin
class StateMachine<Event, Action>(
    callback: ActionCallback<Action>
) {
    
    val callback = WeakReference(callback)
    private var _current: State<Event, Action>? = null
    val current: State<Event, Action>? get() = _current

    fun transition(state: State<Event, Action>) {
        _current?.apply {
            teardown()
            callback = null
        }
        _current = null

        state.callback = callback
        state.setup()
        _current = state
    }

    fun process(event: Event) {
        _current?.process(event) ?: run {
            Log.w("StateMachine", "Event dropped because there is not state to consume it")
        }
    }

}

interface ActionCallback<Action> {

    fun action(action: Action)

}

interface State<Event, Action> {

    var callback: WeakReference<ActionCallback<Action>>?

    suspend fun setup() {}
    suspend fun process(event: Event) {}
    suspend fun teardown() {}

}

abstract class AbstractState<Event, Action>: State<Event, Action> {

    override var callback: WeakReference<ActionCallback<Action>>? = null
    private val _stateScope = object : CoroutineScope {
        override val coroutineContext: CoroutineContext = SupervisorJob() + Dispatchers.Main.immediate
        fun cancel() {
            coroutineContext.cancel()
        }
    }
    protected val stateScope: CoroutineScope get() = _stateScope

    fun action(vararg triggers: Action) {
        callback?.get()?.let { callback ->
            for (trigger in triggers) {
                callback.action(trigger)
            }
        } ?: run {
            Log.e("StateMachine", "No callback available!")
        }
    }

    override fun teardown() {
        _stateScope.cancel()
        super.teardown()
    }

}
```

Now, there are a couple things I'm not happy about this implementation, primarily the ActionCallback interface. It would be far more Kotlin-y and Reactive Programming-y if instead there was a Flow or something to which was subscribed. I'll leave that as an exercise for the reader.

Before we start writing states, it's a good idea to define the ways in which our state machine will interact with the outside world using Events and Actions. As you may have gathered, Events are passed to the State, while Actions are passed back to tell the ViewModel to do something. Let's make a basic one:

```kotlin
sealed interface ExampleEvent {
    object OpenBrowseButtonClicked: ExampleEvent
    object OpenSearchButtonClicked: ExampleEvent
    class SearchTextChanged: ExampleEvent
}

sealed interface ExampleAction {
    class Transition(val state: Class<*>): ExampleAction
    class ShowLoadingState(val show: Boolean): ExampleAction
    // Null here means "hide"
    class ShowErrorState(val exception: Exception?): ExampleAction
    class PresentItems(val results: List<String>): ExampleAction
    class ShowSaveSearchButton(val show: Boolean): ExampleAction
}
```

After we've done that, we can actually start writing our states!

```kotlin
class BrowseExampleState(
    val browseRepository: BrowseRepository
): AbstractState<ExampleEvent, ExampleAction>() {

    suspend fun setup() {
        super.setup()
        load()
    }

    suspend fun process(event: ExampleEvent) {
        super.process(event)
        when (event) {
            is ExampleEvent.OpenSearchButtonClicked -> {
                action(Transition(SearchExampleState::class.java)))
            }
            else -> {}
        }
    }

    private fun load() {
        action(ExampleAction.ShowLoadingState(true))
        stateScope.launch {
            try {
                action(
                    ExampleAction.PresentItems(
                        ExampleAction.browseRepository.load()
                    )
                )
                action(ExampleAction.ShowLoadingState(false))
            } catch(e: Exception) {
                action(
                    ExampleAction.ShowErrorState(
                        e
                    )
                )
                action(ExampleAction.ShowLoadingState(false))
            }
        }
    }

} 

...
```

Wow that was easy! Next we need to actually hook this all up to a ViewModel, which is similarly simple.

```kotlin
class ExampleViewModel(
    val browseRepository: BrowseRepository
): ViewModel(), ActionCallback<ExampleAction> {

    private val stateMachine = StateMachine<ExampleEvent, ExampleAction>(this)

    fun setup() {
        transition(BrowseExampleState::class.java)
    }

    private fun transition(state: Class<*>) {
        stateMachine.transition(
            when(state) {
                is BrowseExampleState::class.java -> BrowseExampleState(
                    browseRepository
                )
            } as State<ExampleEvent, ExampleAction>
        )
    }

    override fun action(action: ExampleAction) {
        when (action) {
            is ExampleAction.Transition -> transition(action.state)
            else -> { /* Implement more handlers here */ }
        }
    }

    fun openSearchButtonClicked() {
        stateMachine.process(ExampleEvent.OpenSearchButtonClicked)
    }

}
```

Wow that was pretty neat! I hope you learned something from this tutorial. Feel free to tell me how this can be improved because I'm sure it can be.

*La Fin*
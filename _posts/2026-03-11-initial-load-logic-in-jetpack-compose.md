---
layout: post
title: "Where Should Initial Load Logic Live in Jetpack Compose?"
date: 2026-03-11
tags: [android, jetpack-compose, kotlin, architecture, viewmodel]
---

*A production decision guide born from 3 LinkedIn posts, 50+ engineers disagreeing, and the humbling realization that everyone was right*

---

## Table of Contents

1. [The Debate That Wouldn't End](#the-debate-that-wouldnt-end)
2. [Why This Is Harder Than It Looks](#why-this-is-harder-than-it-looks)
3. [The Patterns](#the-patterns)
   - [Pattern 0 — The Smell](#pattern-0--the-smell)
   - [Pattern 1 — init {}](#pattern-1--init-)
   - [Pattern 2 — Explicit Action Dispatch](#pattern-2--explicit-action-dispatch-clean-mvvm--toad)
   - [Pattern 3 — .onStart + stateIn](#pattern-3--onstart--onsubscription--statein)
   - [Pattern 4 — The Trigger Pattern](#pattern-4--the-trigger-pattern)
4. [The Decision Framework](#the-decision-framework)
5. [The Meta-Lesson](#the-meta-lesson)
6. [Do You Even Need a ViewModel?](#do-you-even-need-a-viewmodel)
7. [Conclusion](#conclusion)

---

## The Debate That Wouldn't End

I wrote a LinkedIn post calling `LaunchedEffect(Unit) { viewModel.loadData() }` a code smell. The community pushed back hard.

I wrote a follow-up defending `init{}` as a cleaner alternative. The community pushed back harder.

Then I wrote about `.onStart` + `stateIn` as the reactive approach. Same result.

Fifty-plus engineers weighed in across three posts. Some agreed, some disagreed violently, and some proposed entirely different patterns I hadn't considered. I spent more time in the comments than I did writing the posts.

Here's the uncomfortable conclusion I reached: **they were all right.**

Not because every pattern is equally good — some are genuinely better than others in specific contexts — but because the answer depends on constraints that most articles ignore entirely. Your team size. Your testing discipline. Whether you need retry. Whether your screen is reactive-heavy or a simple one-shot load.

Most existing content on this topic falls into one of three camps: "Use `LaunchedEffect(Unit)`" (old, naive), "Move to `init{}`" (overcorrection with no nuance), or "Use `stateIn`" (one-sided, skips the gotchas). Each presents a single pattern as *the* answer.

This article is different. It covers all the real patterns with honest trade-offs, provides an actual decision framework rather than "it depends," and validates everything with production context and community feedback from engineers who do this daily.

---

## Why This Is Harder Than It Looks

Android development has a unique set of constraints that make this question genuinely difficult. It's not that developers are overthinking it — it's that the platform forces you to think about things that don't exist on other platforms.

Here's the fundamental tension: **the ViewModel and the Composable have different lifecycles.**

The ViewModel survives configuration changes. The Composable doesn't. When the user rotates their phone, the Composable is destroyed and recreated from scratch, but the ViewModel stays alive. This is by design — it's the whole point of ViewModel — but it creates a mismatch that every initial-load pattern has to deal with.

![Figure 1.1: Rotation — LaunchedEffect fires again on config change](/assets/images/initial-load-logic/f11.png)
*Figure 1.1: On rotation, the Composable is destroyed and recreated — LaunchedEffect(Unit) fires again.*

![Figure 1.2: Backstack — LaunchedEffect fires again on navigate back](/assets/images/initial-load-logic/f12.png)
*Figure 1.2: Navigate away and back — the Composable re-enters composition and the effect fires again.*

![Figure 1.3: Process death — ViewModel is recreated from scratch](/assets/images/initial-load-logic/f13.png)
*Figure 1.3: After process death, the ViewModel is recreated from scratch. In-memory state is gone.*

The main lifecycle challenges that force trade-offs in every approach:

- **Configuration changes:** The Composable is destroyed and recreated, but the ViewModel survives. Any `LaunchedEffect` will re-fire. Any `init{}` won't.
- **Process death + SavedStateHandle:** The system can kill your app in the background. When the user returns, the ViewModel is recreated from scratch. `SavedStateHandle` is the mechanism for surviving this, but not every pattern works cleanly with it.
- **Backstack:** When the user navigates to another screen, your screen's Composable may leave composition entirely, but the ViewModel stays alive (scoped to the navigation graph). When they come back, the Composable re-enters composition and effects fire again.
- **Unit testing coroutines:** Some patterns make testing straightforward; others require `TestScope`, `advanceUntilIdle()`, and careful coroutine management.

Every pattern is a trade-off against at least one of these constraints. There is no escape.

---

## The Patterns

Each pattern below follows the same structure: what it looks like, why people reach for it, the real gotchas (not just "be careful"), and when it's actually the right choice.

### Pattern 0 — The Smell

```kotlin
@Composable
fun MyScreen(viewModel: MyViewModel = viewModel()) {
    LaunchedEffect(Unit) {
        viewModel.loadData()  // Why here? What triggers this?
    }
    // ... UI
}
```

This is the pattern most tutorials teach. It's also the one that causes the most trouble in production.

**Why people use it:** It's simple. It runs once when the Composable enters composition. It "just works" in the happy path.

**The real gotchas:**

- **It hides business intent behind UI plumbing.** The Composable is now responsible for deciding *when* to load data. That's a business decision masquerading as a lifecycle event.
- **It re-fires on re-entering composition.** Navigate away, come back — `LaunchedEffect(Unit)` fires again. Rotate the device — fires again. The ViewModel may have already loaded the data, but the Composable doesn't know that. You end up with duplicate network calls, race conditions, and state flickering.
- **It's not testable without a Compose test harness.** You can't unit test this in isolation. The load trigger is embedded in the UI layer.
- **No retry or refresh support.** You'd need to add more state management to handle retry, which further tangles the UI and business layers.

**The deeper problem:** `LaunchedEffect(Unit)` is **composition-driven, not UI-driven.** A ViewModel should react to business semantics: "screen opened," "user requested refresh," "navigation argument changed." It should not react to rendering mechanics: "this composable was added to the tree."

**Best for:** Nothing. There is always a better option.

---

### Pattern 1 — init {}

```kotlin
class MyViewModel : ViewModel() {
    init {
        viewModelScope.launch { load() }
    }

    private suspend fun load() {
        // fetch data, update state
    }
}
```

This is the first reflex when someone realizes Pattern 0 is a smell. Move the load into `init{}`, and now the ViewModel owns it. Clean, right?

**Why people like it:** The load happens on ViewModel creation. It survives configuration changes. No Compose-side effects at all. The UI just observes state.

**The real gotchas:**

- **No screen-visibility awareness.** The ViewModel is created when the navigation graph instantiates it, which might happen before the user ever sees the screen.
- **Runs immediately in unit tests.** The moment you instantiate the ViewModel in a test, `init{}` fires. You need `TestScope` and `advanceUntilIdle()` to control timing.
- **You lose conditional load control.** With `init{}`, the load always happens. You can't conditionally skip it based on navigation arguments, screen state, or user intent.
- **No retry or refresh support.** There's no built-in mechanism for retry or pull-to-refresh.

**Best for:** Simple prototypes, one-shot screens with no retry needs, or fast-moving teams that accept the trade-offs.

---

### Pattern 2 — Explicit Action Dispatch (Clean MVVM / TOAD)

```kotlin
// Composable: signals lifecycle, NOT business logic
@Composable
fun MyScreen(viewModel: MyViewModel = viewModel()) {
    LaunchedEffect(Unit) {
        viewModel.onAction(Action.ScreenStarted)
    }

    val uiState by viewModel.uiState.collectAsStateWithLifecycle()
    // ... render uiState
}

// ViewModel: stays idle until told to start
class MyViewModel : ViewModel() {
    private val _uiState = MutableStateFlow<UiState>(UiState.Idle)
    val uiState: StateFlow<UiState> = _uiState.asStateFlow()

    fun onAction(action: Action) {
        when (action) {
            Action.ScreenStarted -> viewModelScope.launch { load() }
            Action.Retry -> viewModelScope.launch { load() }
        }
    }

    private suspend fun load() {
        _uiState.value = UiState.Loading
        // fetch data, update _uiState
    }
}

sealed interface Action {
    data object ScreenStarted : Action
    data object Retry : Action
}
```

Wait — doesn't this use `LaunchedEffect(Unit)`, the thing I just called a smell?

Yes. But the distinction matters: **Pattern 0 uses LaunchedEffect as a business trigger.** Pattern 2 uses it as a **lifecycle signal.** The Composable isn't deciding what to load or when — it's signaling "the screen is now visible." The ViewModel decides what to do with that signal.

**Why people like it:**

- **Explicit and auditable.** Every state change traces back to a named action.
- **Trivially testable.** In unit tests, you call `viewModel.onAction(Action.ScreenStarted)` directly. No Compose runtime, no `LaunchedEffect`, no test harness.
- **Works cleanly with `SavedStateHandle.toRoute<>()`.**
- **First-class retry and refresh.** Adding `Action.Retry` or `Action.PullToRefresh` is a one-line addition to the `when` block.

**The real gotchas:**

- **More ceremony than `init{}`.** For a simple screen, this can feel like overkill.
- **The ceremony is worth it.** Once your app has more than a handful of screens, the consistency and testability pay for themselves many times over.

A stricter variant is **TOAD** (Typed Object Action Dispatch), where the ViewModel dispatches typed events to external handler classes. If your team follows MVI principles, TOAD is Pattern 2 taken to its logical extreme.

**Best for:** Most production apps. This is the default recommendation unless you have a specific reason to choose otherwise.

---

### Pattern 3 — .onStart / .onSubscription + stateIn

```kotlin
class MyViewModel : ViewModel() {
    val uiState: StateFlow<UiState> = repository.getDataFlow()
        .onStart { emit(UiState.Loading) }
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5_000),
            initialValue = UiState.Loading
        )
}
```

This is the reactive purist's answer: no `LaunchedEffect` in the Composable at all. The load starts when the UI subscribes to the `StateFlow`, and stops when the screen is gone for longer than 5 seconds. The ViewModel is a pure reactive pipeline.

**Why people like it:**

- **Zero Compose side effects.** The Composable just collects state.
- **Starts exactly when the UI observes.** If nobody is watching, nothing happens.
- **Stops when the screen is gone.** `WhileSubscribed(5_000)` means the upstream flow is cancelled 5 seconds after the last subscriber disappears.
- **Fits naturally into reactive chains.**

**The real gotchas:**

- **Hot→cold→hot semantics can surprise you.** When the timeout expires, the flow goes cold. When a new subscriber appears, `.onStart` fires again — the load can re-trigger.
- **No built-in retry.** You can't emit to a cold flow from outside. If the user taps "retry," you need a separate mechanism.
- **Test timing subtleties.** You need `advanceUntilIdle()` to let the flow pipeline settle in unit tests.

**Best for:** Reactive-heavy screens with multiple upstream flows, teams allergic to any Compose side effects, and ViewModels that are pure transformation pipelines.

---

### Pattern 4 — The Trigger Pattern

```kotlin
class MyViewModel : ViewModel() {
    private val retryTrigger = MutableSharedFlow<Unit>(extraBufferCapacity = 1)

    val uiState: StateFlow<UiState> = retryTrigger
        .onStart { emit(Unit) }  // auto-trigger on first subscription
        .transformLatest {
            emit(UiState.Loading)
            val result = repository.loadData()
            emit(result.fold(
                onSuccess = { UiState.Success(it) },
                onFailure = { UiState.Error(it.message) }
            ))
        }
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5_000),
            initialValue = UiState.Loading
        )

    fun retry() {
        retryTrigger.tryEmit(Unit)
    }
}
```

This is Pattern 3's big sibling. It combines the reactive benefits of `.onStart + stateIn` with first-class retry and refresh support — all in a single reactive pipeline.

**Why people like it:**

- **Single pipeline for everything.** Initial load, retry, and pull-to-refresh all flow through the same `transformLatest` block.
- **`transformLatest` cancels in-flight calls.** If the user taps retry while a previous load is still running, the previous coroutine is cancelled automatically. No race conditions.
- **No `LaunchedEffect` in Compose.**
- **Declarative.** The entire load lifecycle is expressed as a flow transformation.

**The real gotchas:**

- **More complex mental model.** If your team isn't comfortable with `transformLatest`, `SharedFlow`, and reactive flow operators, this pattern has a steeper learning curve than Pattern 2.
- **Don't add extra `onStart` on the outer chain.** A common mistake is adding `.onStart { emit(UiState.Loading) }` on the outer chain after `transformLatest`, which creates duplicate emissions.

**Best for:** Screens that need retry, pull-to-refresh, or periodic reload, combined with reactive upstream flows.

---

## The Decision Framework

![Figure 2: Decision flowchart — which pattern to use](/assets/images/initial-load-logic/f2.png)
*Figure 2: Decision flowchart. Start at the top — four yes/no questions lead you to the right pattern.*

**Does your screen need to load data when it appears?** Yes → continue below.

**Q1: Do you need retry / pull-to-refresh / multiple reload triggers?**
- YES → **Pattern 2** (explicit action dispatch) or **Pattern 4** (trigger-based reactive).
- NO → continue to Q2.

**Q2: Is your screen reactive-heavy?** (multiple upstream flows, `.combine`, `.flatMapLatest`, `SavedStateHandle.getStateFlow`)
- YES → **Pattern 3** (`.onStart` + `stateIn`). Fits naturally into the reactive chain.
- NO → continue to Q3.

**Q3: Does your team care about strict auditability?**
- YES → **Pattern 2** with MVVM or TOAD.
- NO → continue to Q4.

**Q4: Is this a simple, one-shot load with no retry, on a small or fast-moving team?**
- YES → **Pattern 1** (`init{}`). Accept the trade-offs.
- NO → Default to **Pattern 2**.

### The Comparison Table

| Pattern | Testable | Ghost-free | SavedState | Retry | Ceremony | Best for |
|---|:---:|:---:|:---:|:---:|:---:|---|
| `LaunchedEffect(Unit)` → VM directly | ❌ | ❌ | ❌ | ❌ | Low | nothing — avoid |
| `init {}` in ViewModel | ⚠️ | ⚠️ | ✅ | ❌ | Low | prototypes, simple screens |
| Explicit Action Dispatch (MVVM/TOAD) | ✅ | ✅ | ✅ | ✅ | Medium | most production apps |
| `.onStart` / `.onSubscription` + `stateIn` | ✅ | ✅ | ✅ | ⚠️ | Low | reactive screens, flow-heavy VMs |
| Trigger (`SharedFlow` + `transformLatest`) | ✅ | ✅ | ✅ | ✅ | Medium | retry-heavy screens |

**Ghost-free** = no network calls when the screen is off-screen or in the backstack.  
**SavedState** = works cleanly with `SavedStateHandle` / `toRoute<>()`.

---

## The Meta-Lesson

The real code smell was never `LaunchedEffect` itself.

It was **using the UI layer as an imperative business trigger.**

The fix isn't a single pattern — it's a clear contract:

- **ViewModel** owns state and logic.
- **Composable** observes state and signals lifecycle.

Once that line is clear, even `LaunchedEffect(Unit)` can be clean — as long as it's sending a lifecycle signal (like `Action.ScreenStarted`), not triggering business logic directly.

---

## Do You Even Need a ViewModel?

This was the sharpest critique in the entire series. The honest answer:

**The case for keeping the ViewModel:**
- Config change survival. The ViewModel lives across rotation; the Composable doesn't.
- Process death + SavedStateHandle. Reactive flows alone don't serialize state across process death.
- Testability boundary. The ViewModel is the seam where you swap real repositories for fakes in unit tests.
- Shared state across multiple Composables on the same screen.

**The case for questioning it:**
- For truly stateless, read-only screens with no side effects, a ViewModel is ceremony.
- A plain `produceState` or a Compose-aware library like Molecule or Circuit can be cleaner for simple cases.

Patterns 3 and 4 don't strip the ViewModel of purpose — they keep load logic *inside* it, just triggered reactively rather than imperatively. The question is only about *how* it decides to start work, not *whether* it owns state.

---

## Conclusion

There is no silver bullet. Android's constraints — configuration changes, process death, backstack, recomposition — guarantee that every pattern has trade-offs.

But "it depends" is not good enough. That's why the decision framework exists: it turns a vague architectural question into four concrete yes/no checks that point you to a specific pattern.

If you're unsure, start with Pattern 2. The explicit action dispatch approach works for most production apps, scales cleanly, and gives you a clear path to Pattern 4 if you need reactive retry later. The ceremony is real, but it pays for itself the first time you debug a load-related bug and can trace exactly what triggered it.

Thanks to everyone who pushed back in the comments across this series — Sevban, Dmitrii, Emin, Hristijan, Colin, Adrian, and many others. You made this guide better than anything I could have written alone.

---

## References

**Android / Jetpack**
- [Jetpack Compose](https://developer.android.com/jetpack/compose)
- [ViewModel](https://developer.android.com/topic/libraries/architecture/viewmodel)
- [SavedStateHandle](https://developer.android.com/topic/libraries/architecture/viewmodel/viewmodel-savedstate)
- [StateFlow / SharedFlow](https://developer.android.com/kotlin/flow/stateflow-and-sharedflow)
- [LaunchedEffect](https://developer.android.com/develop/ui/compose/side-effects#launchedeffect)
- [collectAsStateWithLifecycle](https://developer.android.com/reference/kotlin/androidx/lifecycle/compose/package-summary)
- [Navigation Compose](https://developer.android.com/jetpack/compose/navigation)

**Alternative Architecture Libraries**
- [Molecule](https://github.com/cashapp/molecule) (Cash App)
- [Circuit](https://github.com/slackhq/circuit) (Slack)
- [TOAD — Typed Object Action Dispatch](https://medium.com/@aumaidkh/toad-a-kotlin-first-architecture-pattern-that-finally-made-my-viewmodels-boring-b615a9ab6c30)
- [Orbit MVI](https://github.com/orbit-mvi/orbit-mvi)
- [MVIKotlin](https://github.com/arkivanov/MVIKotlin)

---

*This article grew out of a 4-part LinkedIn series.*

*Follow me on [Medium](https://medium.com/@sergey.neskoromny) and [LinkedIn](https://www.linkedin.com/in/sergey-neskoromny/) for updates.*

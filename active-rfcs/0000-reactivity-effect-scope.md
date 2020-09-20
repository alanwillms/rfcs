- Start Date: 2020-08-20
- Target Major Version: 3.x
- Reference Issues: (fill in existing related issues, if any)
- Implementation PR: (leave this empty)

# Summary

A new `effectScope` API for `@vue/reactivity` that automatically collects the effects in a function call, and returns a stop handler to clean them up.

# Basic example

```ts
// effect, computed, watch, watchEffect created inside the scope will be collected

const scope = effectScope(() => {
  const doubled = computed(() => counter.value * 2)

  watch(doubled, () => console.log(double.value))

  watchEffect(() => console.log('Count: ', double.value))
})

// to dispose all effects in the scope
stop(scope)
```

A scope will be defined as

```ts
interface EffectScope {
  uid: number
  active: boolean
  effects: (ReactiveEffect | EffectScope)[]
}
```

# Motivation

In Vue's component `setup()`, effects will be collected and bound to the current instance. When the instance get unmounted, effects will be disposed automatically. This is a convenient and intuitive feature.

However, when we are using them out side of components or as a standalone package, it's not that simple. For example, this might be what we need to do for disposing the effects of `computed` & `watch`

```ts
const disposables = []

const counter = ref(0)
const doubled = computed(() => counter.value * 2)

disposables.push(() => stop(doubled.effect))

const stopWatch1 = watchEffect(() => {
  console.log(`counter: ${counter.value}`)
})

disposables.push(stopWatch1)

const stopWatch2 = watch(doubled, () => {
  console.log(double.value)
})

disposables.push(stopWatch2)
```

And to stop the effects:

```ts
disposables.forEach(f => f())
disposables = []
```

Especially when we have some long and complex composable code, it's laborious to manually collect all the effects . It's also easy to forget collecting them (or you don't have access to effects created in the composable functions) which might result memory leakage and unexpected behavior.

# Detailed design

Similar to what we did for `currentInstance`, on effect creation, if the `activeEffectScope` is presented, push itself into the scope.

```ts
function createReactiveEffect( /* ... */ ) {
  const effect = function reactiveEffect(): unknown {
    /* ... */ 
  } as ReactiveEffect
  /* ... */
  
  if (activeEffectScope)
    activeEffectScope.effects.push(effect) // <-
  
  return effect
}
```

### Nested Scopes

Nested scopes should also be collected by its parent scope. And when the parent scope get disposed, all its descendant scopes will be also stopped.

```ts
const scope = effectScope(() => {
  const doubled = computed(() => counter.value * 2)
  
  // not need to get the stop handler, it will be collected by the outer scope
  effectScope(() => {
    watch(doubled, () => console.log(double.value))
  })

  watchEffect(() => console.log('Count: ', double.value))
})

// to dispose all effects, as well as the nested scopes
stop(scope)
```

### Implementation

I have implemented a third-party library [@vue-reactivity/scope](https://github.com/vue-reactivity/scope) which aims to solve the exact same problem. However, since we have no way to get access the internal states of effects, [it has to wrap & re-export](https://github.com/vue-reactivity/scope/blob/master/src/hijack.ts#L11) the APIs (`effect` and `computed` etc.) to make the auto collecting work. This deviated from the Vue's design and can not be directly reused for Vue applications. As this is a quite low-level feature, I think it's better to be done in `@vue/reactivity` itself. And could benefit Vue itself and the ecosystem built on top of `@vue/reactivity`.

### Impact to Vue

In `@vue/runtime-dom`, [it wraps the `computed` to add the instance binding](https://github.com/vuejs/vue-next/blob/master/packages/runtime-core/src/apiComputed.ts). This make the following statements NOT equivalent

```ts
import { computed } from '@vue/reactivity'
import { computed } from 'vue'
```  

This should not be an issue for most of the users, but for some libraries that would like to only rely on `@vue/reactivity` (for more flexible usages) this my cause some minor difference.

With this RFC, `@vue/runtime-dom` might also be able to do some refactoring by using the `effectScope` to collect those effects transparently. So the computed wrapping might not be necessary anymore.

# Drawbacks

- It doesn't work well with async scope function
- Variables created in the closure will be harder to access from the outer closure (or should we accept it's return value and pass it out?)

# Alternatives

Maybe an `onEffectCreation` hook instead, which will be called on every effect creation. This can provide more flexibility and even let users create their own scope utils or custom frameworks' lifecycles.


```ts
onEffectCreation((effect) => {
  // custom function
  
  // for example, in runtime-core, we can use this to collect the effect
  recordInstanceBoundEffect(effect)
})
```

```ts
function createReactiveEffect( /* ... */ ) {
  /* ... */
  
  invokeHook('effectCreation', effect)
  
  return effect
}
```

# Adoption strategy

This is a new API and should not affect existing code.

# Unresolved questions

- Naming
- Migration & Refactoring in `@vue/runtime-dom`
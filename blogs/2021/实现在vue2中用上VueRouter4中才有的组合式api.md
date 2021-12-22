# 实现在 Vue2 中用上 Vue Router 4 中才有的组合式 API

> 在 vue3 以及 Vue Router 4 中，可以使用组合式 API 的方式来操作路由，而基于 vue2 的 Vue Router 3 并没有提供相应的能力。想要在 vue2 项目中愉快的玩耍组合式 API，就必须解决这个问题。

## Vue Router 4 提供了哪些组合式 API

首先，让我们先看看 Vue Router 4 都提供了哪些组合式 API：

> https://next.router.vuejs.org/zh/guide/advanced/composition-api.html

1. 在 setup 中访问路由和当前路由
    1. useRoute
    2. useRouter
2. 导航守卫
    1. onBeforeRouteLeave
    2. onBeforeRouteUpdate
3. useLink

## useRoute & useRouter

这两个组合式 API 其实就相当于之前使用的 `this.$route` & `this.$router`，由于在 `setup` 中无法使用 `this`，只能通过 [getCurrentInstance](https://v3.cn.vuejs.org/api/composition-api.html#getcurrentinstance) 去访问。

乍看上去，这似乎是最容易封装的两个组合式 API，以 `useRouter` 为例：

```ts
export const useRouter = () => getCurrentInstance()?.proxy.$router as VueRouter;
```

但在使用 `useRoute` 的过程中，却发现在以下这个例子中，无法监听到参数的变化：

```ts
const tid = ref(route.params.id);
watch(() => route.params, params => tid.value = params.id);
```

在 [@vue/composition-api#809](https://github.com/vuejs/composition-api/issues/809) 中，同样有人提出了这个问题。[antfu](https://github.com/antfu) 回复，应当使用 `return computed(() => vm.proxy.$route)`。即直接返回 $route 确实会丢失响应性。

后续的回复中，也有人给出了[一个巧妙的方案](https://github1s.com/ambit-tsai/vue2-helpers/blob/HEAD/src/vue-router.ts)以消除 `.value`：

```ts
export function useRoute() {
    const router = useRouter()
    if (!currentRoute) {
        const routeRef = shallowRef({
            path: '/',
            name: undefined,
            params: {},
            query: {},
            hash: '',
            fullPath: '/',
            matched: [],
            meta: {},
            redirectedFrom: undefined,
        } as Route);
        const computedRoute = {} as {
            [key in keyof Route]: ComputedRef<Route[key]>
        }
        for (const key of Object.keys(routeRef.value) as (keyof Route)[]) {
            computedRoute[key] = computed<any>(() => routeRef.value[key])
        }
        router.afterEach(to => routeRef.value = to)
        currentRoute = reactive(computedRoute)
    }
    return currentRoute
}
```

但实际上，该方法确实保持了与 Vue Router 4 中一致的用法。但偶尔依旧会丢失响应性。经过多次尝试，终于发现问题出在第一次调用 `useRoute` 的组件被销毁后，响应性就丢失了！

### 副作用作用域

想要解决这个问题，需要了解一个概念——[Effect 作用域](https://github.com/vuejs/rfcs/blob/master/active-rfcs/0041-reactivity-effect-scope.md#affected-usage-in-vue-core)。

我们都知道，虽然大部分时候不会去使用，但 `watchEffect` 是有返回值的：

```ts
function watchEffect(
  effect: (onInvalidate: InvalidateCbRegistrator) => void,
  options?: WatchEffectOptions
): StopHandle
```

在某些情况下，我们是可以[主动终止副作用的监听](https://v3.cn.vuejs.org/guide/reactivity-computed-watchers.html#%E5%81%9C%E6%AD%A2%E4%BE%A6%E5%90%AC)的。

```ts
const stop = watchEffect(() => console.log(count.value))
stop()
```

之所以我们大部分时候不会在意这一点，自然是因为大部分副作用贯穿了组件的整个生命周期，不需要终止。而就像在组合式 API 出现前的那样，声明的 `watch` 在组件销毁时自动终止，计算属性也是如此。

如此，就解释了为何第一次调用 `useRoute` 的组件被销毁后，响应性就丢失了——这些计算属性，是在第一次调用 `useRoute` 的组件中调用的，在其销毁后也停止了副作用监听。

自然，我们可以使用 [Effect 作用域 API](https://v3.cn.vuejs.org/api/effect-scope.html#effectscope) 来解决这个问题，但还有更通用的方法。

### createGlobalState

[vueuse](https://vueuse.org/) 是一个很棒的库，其中就有一个解决此类问题的良方 [createGlobalState](https://github1s.com/vueuse/vueuse/blob/main/packages/shared/createGlobalState/index.ts)。

其本质就是一个单例+独立副作用域，完美的符合了我们需要，`useRoute` 改造如下：

```ts
export const useRoute = createGlobalState(() => {
  const router = useRouter();
  const routeRef = shallowRef<Route>({
    path: '/',
    name: undefined,
    params: {},
    query: {},
    hash: '',
    fullPath: '/',
    matched: [],
    meta: {},
    redirectedFrom: undefined,
  });
  const computedRoute = useMemoRefs(routeRef => routeRef, routeRef);
  routeRef.value = getCurrentInstance()?.proxy.$route as Route;
  router.afterEach(to => routeRef.value = to);
  return reactive(computedRoute);
});
```

## 导航守卫

实现导航守卫的本质是一个很通用的问题——自定义组件生命周期的实现。说到底，导航守卫和 `onMounted` 没什么区别。所以我们只要了解了 [`@vue/composition-api` 是如何给 vue2 实现 `onMounted` 的](https://github1s.com/vuejs/composition-api/blob/HEAD/src/apis/lifecycle.ts)，我们也就了解了如何去实现一个导航守卫。

可以与之前提到的 [vue2-helpers](https://github1s.com/ambit-tsai/vue2-helpers/blob/HEAD/src/vue-router.ts) 做个对比，两者的实现思路差异不大，区别只是导航守卫需要挂到实例的原型链上才会生效。

在稍加改动 `@vue/composition-api` 的 `createLifeCycle` 之后，我们就能轻而易举的实现导航守卫了：

```ts
export const onBeforeRouteLeave = createLifeCycle('beforeRouteLeave', (cb: (to: Route, from: Route) => MaybePromise<Parameters<NavigationGuardNext>[0]>) => async (to: Route, from: Route, next: NavigationGuardNext) => {
  next(await cb(to, from));
});
export const onBeforeRouteUpdate = createLifeCycle('beforeRouteUpdate', (cb: (to: Route, from: Route) => MaybePromise<Parameters<NavigationGuardNext>[0]>) => async (to: Route, from: Route, next: NavigationGuardNext) => {
  next(await cb(to, from));
});
```

## useLink

Vue Router 4 将 RouterLink 的内部行为作为一个组合式 API 函数公开。它提供了与 v-slot API 相同的访问属性。

这其实在 [vueuse](https://vueuse.org/) 中有大量类似的例子，且 Vue Router v3 与 v4 中 RouterLink 的参数有一些差异，完全实现一个没有什么意义。这里只实现了一个简易版本的，支持了常用的几个参数：

```ts
export const useLink = (props: MaybeRefs<UseLinkOptions>) => {
  const $router = useRouter();
  const { location, route, href, handler } = useMemoRefs(({ to, append, replace }) => {
    const { location, route, href } = $router.resolve(
      to,
      $router.currentRoute,
      append,
    );
    const handler = (location: Location) => (replace ? $router.replace(location) : $router.push(location));
    return { location, route, href, handler };
  }, props);

  const navigate = () => handler.value(location.value);
  return {
    href,
    route,
    navigate,
  };
};
```

在有了此类组合式 API 后，实现 RouterLink 这样的组件就相当容易了（此处仅为举例）：

```ts
export const RouterLink = defineComponent({
  props: { ... },
  setup(props, { slots }) {
    const { navigate: click } = useLink()
    ...
    return () => {
      return h(props.as ?? 'a', { on: { click } }, [slots.default()])
    };
  },
});
```

在实际的实现过程中，应该是先有 `useLink` 后有 `RouterLink`，在有 `RouterLink` 的情况下重写一个 `useLink` 意义不大。

## 参考

+ https://next.router.vuejs.org/zh/guide/advanced/composition-api.html
+ https://v3.cn.vuejs.org/api/composition-api.html#getcurrentinstance
+ https://github.com/vuejs/composition-api/issues/809
+ https://github1s.com/ambit-tsai/vue2-helpers/blob/HEAD/src/vue-router.ts
+ https://github.com/vuejs/rfcs/blob/master/active-rfcs/0041-reactivity-effect-scope.md#affected-usage-in-vue-core
+ https://v3.cn.vuejs.org/api/effect-scope.html#effectscope
+ https://vueuse.org/
+ https://github1s.com/vueuse/vueuse/blob/main/packages/shared/createGlobalState/index.ts
+ https://github1s.com/vuejs/composition-api/blob/HEAD/src/apis/lifecycle.ts

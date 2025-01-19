# useState のしくみ

- [ソースコード](https://github.com/facebook/react)からちゃんとわかりたい
  - 今回は`packeges/react/src/ReactHooks.js`あたりを見ることになると思う

### Hooks のしくみ

`packeges/react/src/ReactHooks.js`にて、以下のように定義されている。

```javascript
export function useState<S>(
  initialState: (() => S) | S
): [S, Dispatch<BasicStateAction<S>>] {
  const dispatcher = resolveDispatcher();
  return dispatcher.useState(initialState);
}

export function useEffect(
  create: () => (() => void) | void,
  deps: Array<mixed> | void | null
): void {
  const dispatcher = resolveDispatcher();
  return dispatcher.useEffect(create, deps);
}

...
```

つまり、

1. Dispatcher の解決
2. 現在の dispatcher で Hooks を実行

を行う。

#### dispatcher とは

- React 内部で現在のフックの状態を管理・提供するための中心的なオブジェクト。
- レンダリング中のコンポーネントに基づいて、適切なフックを提供する。つまりはグローバルなフックの管理ツール的な感じ！

#### Dispatcher の解決とは

- 現在にて適切なフックを取得するってこと。
- 実装は`resolveDispatcher()`関数にて、以下のような感じ。

```javascript
function resolveDispatcher() {
  const dispatcher = ReactCurrentDispatcher.current;
  return ((dispatcher: any): Dispatcher);
}
```

- いま（2025/01/19）ソースコード見ると以下だった。

```javascript
function resolveDispatcher() {
  const dispatcher = ReactSharedInternals.H;
  if (__DEV__) {
    if (dispatcher === null) {
      console.error(
        "Invalid hook call. Hooks can only be called inside of the body of a function component. This could happen for" +
          " one of the following reasons:\n" +
          "1. You might have mismatching versions of React and the renderer (such as React DOM)\n" +
          "2. You might be breaking the Rules of Hooks\n" +
          "3. You might have more than one copy of React in the same app\n" +
          "See https://react.dev/link/invalid-hook-call for tips about how to debug and fix this problem."
      );
    }
  }
  // Will result in a null access error if accessed outside render phase. We
  // intentionally don't throw our own error because this is in a hot path.
  // Also helps ensure this is inlined.
  return ((dispatcher: any): Dispatcher);
}
```

### （参考）React 全体のしくみ

1. 状態の流れ: 状態は `hook.memorizedState`に保持される。`dispatch`で更新する。
2. 更新の仕組み: 更新は `queue.pending` にアクションを蓄積し、レンダリング時に計算する。
3. Fiber との関係: 状態やキューはすべて Fiber ツリーに紐づけられている。再レンダリングのスケジューリングに関わる。

### Dispatcher の解決を詳しく見る

Dispatcher の解決とは、現在のフック（を持つ dispatcher）を取得することだった。
それは、`resolveDispatcher`関数で返される`ReactSharedInternals.H` (正確には`ReactSharedInternals`にある`ReactCurrentDispatcher.current`) であった。
では、`ReactSharedInternals.H` はどのように計算されるのか？を見ていきたい

1. `ReactSharedInternals`は React 内で共有されるオブジェクト。`ReactSharedInternals.H`を通じて、`ReactCurrentDispatcher.current`にディスパッチャが格納されている。

2. レンダリングフェーズにて、適切なディスパッチャが設定される（まだ useState が実行される前。useState はこの正しいディスパッチャを元に実行されるもの）。`packages/react/src/ReactFiberHooks.js`の`renderWithHooks`関数にて実行される。これで、Dispatcher は解決した。

3. すべて解決（つまりレンダリングが完了）すると、React は`ReactCurrentDispatcher.current`を`null`または初期状態にリセットする。これにより、不適切な場所でフックが呼び出された場合にエラーが発生するようになる。

#### `renderWithHooks`をくわしくみる

いま（2025/01/19）の実装は以下。

```javascript
export function renderWithHooks<Props, SecondArg>(
  current: Fiber | null,
  workInProgress: Fiber,
  Component: (p: Props, arg: SecondArg) => any,
  props: Props,
  secondArg: SecondArg,
  nextRenderLanes: Lanes
): any {
  renderLanes = nextRenderLanes;
  currentlyRenderingFiber = workInProgress;

  if (__DEV__) {
    hookTypesDev =
      current !== null
        ? ((current._debugHookTypes: any): Array<HookType>)
        : null;
    hookTypesUpdateIndexDev = -1;
    // Used for hot reloading:
    ignorePreviousDependencies =
      current !== null && current.type !== workInProgress.type;

    warnIfAsyncClientComponent(Component);
  }

  workInProgress.memoizedState = null;
  workInProgress.updateQueue = null;
  workInProgress.lanes = NoLanes;

  // The following should have already been reset
  // currentHook = null;
  // workInProgressHook = null;

  // didScheduleRenderPhaseUpdate = false;
  // localIdCounter = 0;
  // thenableIndexCounter = 0;
  // thenableState = null;

  // TODO Warn if no hooks are used at all during mount, then some are used during update.
  // Currently we will identify the update render as a mount because memoizedState === null.
  // This is tricky because it's valid for certain types of components (e.g. React.lazy)

  // Using memoizedState to differentiate between mount/update only works if at least one stateful hook is used.
  // Non-stateful hooks (e.g. context) don't get added to memoizedState,
  // so memoizedState would be null during updates and mounts.
  if (__DEV__) {
    if (current !== null && current.memoizedState !== null) {
      ReactSharedInternals.H = HooksDispatcherOnUpdateInDEV;
    } else if (hookTypesDev !== null) {
      // This dispatcher handles an edge case where a component is updating,
      // but no stateful hooks have been used.
      // We want to match the production code behavior (which will use HooksDispatcherOnMount),
      // but with the extra DEV validation to ensure hooks ordering hasn't changed.
      // This dispatcher does that.
      ReactSharedInternals.H = HooksDispatcherOnMountWithHookTypesInDEV;
    } else {
      ReactSharedInternals.H = HooksDispatcherOnMountInDEV;
    }
  } else {
    ReactSharedInternals.H =
      current === null || current.memoizedState === null
        ? HooksDispatcherOnMount
        : HooksDispatcherOnUpdate;
  }

  // In Strict Mode, during development, user functions are double invoked to
  // help detect side effects. The logic for how this is implemented for in
  // hook components is a bit complex so let's break it down.
  //
  // We will invoke the entire component function twice. However, during the
  // second invocation of the component, the hook state from the first
  // invocation will be reused. That means things like `useMemo` functions won't
  // run again, because the deps will match and the memoized result will
  // be reused.
  //
  // We want memoized functions to run twice, too, so account for this, user
  // functions are double invoked during the *first* invocation of the component
  // function, and are *not* double invoked during the second incovation:
  //
  // - First execution of component function: user functions are double invoked
  // - Second execution of component function (in Strict Mode, during
  //   development): user functions are not double invoked.
  //
  // This is intentional for a few reasons; most importantly, it's because of
  // how `use` works when something suspends: it reuses the promise that was
  // passed during the first attempt. This is itself a form of memoization.
  // We need to be able to memoize the reactive inputs to the `use` call using
  // a hook (i.e. `useMemo`), which means, the reactive inputs to `use` must
  // come from the same component invocation as the output.
  //
  // There are plenty of tests to ensure this behavior is correct.
  const shouldDoubleRenderDEV =
    __DEV__ && (workInProgress.mode & StrictLegacyMode) !== NoMode;

  shouldDoubleInvokeUserFnsInHooksDEV = shouldDoubleRenderDEV;
  let children = __DEV__
    ? callComponentInDEV(Component, props, secondArg)
    : Component(props, secondArg);
  shouldDoubleInvokeUserFnsInHooksDEV = false;

  // Check if there was a render phase update
  if (didScheduleRenderPhaseUpdateDuringThisPass) {
    // Keep rendering until the component stabilizes (there are no more render
    // phase updates).
    children = renderWithHooksAgain(
      workInProgress,
      Component,
      props,
      secondArg
    );
  }

  if (shouldDoubleRenderDEV) {
    // In development, components are invoked twice to help detect side effects.
    setIsStrictModeForDevtools(true);
    try {
      children = renderWithHooksAgain(
        workInProgress,
        Component,
        props,
        secondArg
      );
    } finally {
      setIsStrictModeForDevtools(false);
    }
  }

  finishRenderingHooks(current, workInProgress, Component);

  return children;
}
```

##### 引数

- current: Fiber | null
  - 現在の Fiber (前回のレンダリング時の情報)
- workInProgress: Fiber
  - 今回のレンダリングで処理中の Fiber
- Component: (p: Props, arg: SecondArg) => any
  - コンポーネントの実際の関数
- props: Props
  - コンポーネントの props
- secondArg: SecondArg
  - コンポーネントに渡される追加の引数（通常は context など）
- nextRenderLanes: Lanes
  - レンダリングの優先度

##### 1. レンダリング前の準備

```javascript
renderLanes = nextRenderLanes; // 現在のレンダリング優先度をグローバル変数に設定
currentlyRenderingFiber = workInProgress; // 今回のレンダリング対象であるFiberを設定。これにより、フックが正しいコンポーネントと紐づけられる。（たぶん、フックはグローバルで管理してるから、どのコンポーネントのフックかわからないため。）
```

##### 2. 開発モードでの初期化

```typescript
if (__DEV__) {
  hookTypesDev =
    current !== null
      ? ((current._debugHookTypes: any): Array<HookType>)
      : null;
  hookTypesUpdateIndexDev = -1;
  ignorePreviousDependencies =
    current !== null && current.type !== workInProgress.type;

  warnIfAsyncClientComponent(Component);
}
```

- `hookTypesDev`: 現在のフックの種類を記録
- `ignorePreviousDependencies`: コンポーネントが更新された場合、依存関係の ☑ を無効化
- `warnIfAsyncClientComponent`: 非同期コンポーネントに関する警告の表示

##### 3. Fiber の初期化

```typescript
workInProgress.memoizedState = null;
workInProgress.updateQueue = null;
workInProgress.lanes = NoLanes;
```

- `memorizedState`: フックの状態をリセット
- `updateQueue`: レンダリング中に発生する更新をリセット
- `lanes`: Fiber に割り当てられた優先度をリセット

##### 4. 適切なディスパッチャの更新（ここがだいじ）

```typescript
if (__DEV__) {
  if (current !== null && current.memoizedState !== null) {
    ReactSharedInternals.H = HooksDispatcherOnUpdateInDEV;
  } else if (hookTypesDev !== null) {
    ReactSharedInternals.H = HooksDispatcherOnMountWithHookTypesInDEV;
  } else {
    ReactSharedInternals.H = HooksDispatcherOnMountInDEV;
  }
} else {
  ReactSharedInternals.H =
    current === null || current.memoizedState === null
      ? HooksDispatcherOnMount
      : HooksDispatcherOnUpdate;
}
```

- `ReactSharedInternals.H`にディスパッチャを更新
  - 初期レンダリング時（`current === null || current.memoizedState === null`）: `HooksDispatcherOnMount`関数を実行
  - 更新レンダリング時: `HooksDispatcherOnUpdate`関数を実行

##### 5. コンポーネントの実行

```typescript
let children = __DEV__
  ? callComponentInDEV(Component, props, secondArg)
  : Component(props, secondArg);
```

- `Component`関数: コンポーネントを実行。子要素を生成。
- 開発時: `callComponentInDEV`関数（`Component`関数）の開発時版を実行。

##### 6. レンダリング中の更新

```typescript
if (didScheduleRenderPhaseUpdateDuringThisPass) {
  children = renderWithHooksAgain(workInProgress, Component, props, secondArg);
}
```

- フックの状態がレンダリング中に更新された場合（`didScheduleRenderPhaseUpdateDuringThisPass`）: 再度レンダリングを試みる
- 再レンダリングが安定するまで繰り返す

##### 7. Strict モードでの処理

```typescript
if (shouldDoubleRenderDEV) {
  setIsStrictModeForDevtools(true);
  try {
    children = renderWithHooksAgain(
      workInProgress,
      Component,
      props,
      secondArg
    );
  } finally {
    setIsStrictModeForDevtools(false);
  }
}
```

- 開発モードの StrictMode では、コンポーネントを 2 回実行して副作用の検出を行います。
- 副作用が正しく管理されているかをテストします。

##### 8. フックのレンダリング後の処理

```typescript
finishRenderingHooks(current, workInProgress, Component);
```

- レンダリング中に使用されたすべてのフックの状態や副作用を確定します。
- フックの順序や依存関係が適切かもチェック。

##### 9. 子要素を返す

```typescript
return children;
```

- コンポーネントの出力（主に React 要素）を返す

### これから調べたいこと

- `renderWithHooks`内の主要な関数について調べたい
  - `HooksDispatcherOnMount`
  - `HooksDispatcherOnUpdate`
  - `Conponent`
  - `renderWithHooksAgain`
  - `finishRenderingHooks`
- dispatcher 解決後に実行する関数`dispatcher.useState()`関数を調べたい
- レンダーフェーズ、Fiber について調べたい

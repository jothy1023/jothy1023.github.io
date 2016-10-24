---
title: Vuex 入门
date: 2016-10-05 13:17:48
tag: [Vue, Vuex]
---

### Vuex 是一个专门为 Vue.js 应用所设计的集中式状态管理架构 .  
背景：小型应用里的每个组件维护着自有的状态，即当前应用的状态的一部分，所以整个应用的状态被分散在了各个角落，但是我们经常遇到要把**状态的一部分**共享给多个组件的情况。
> 状态其实可以形象地想成我们的 data 里面的各个属性。

------
#### State
Vuex 使用了单状态树（single state tree），一个 store 对象就存储了整个应用层的状态。它让我们可以更方便地定位某一具体的状态，并且在调试时能简单地获取到当前整个应用的快照。 
- 先埋个伏笔。Vuex 使用的这种 single state tree 与 modularity 模块化是不冲突的，问题是，如何将 state 与 mutation 分到子模块中？
- 要使用 store ，首先必须`Vue.use(Vuex)`，然后将 store `const store = new Vuex.store()` inject 定义到 Vue 实例 app 中`new Vue({store})`，实现从根组件注入到所有子组件中，接着就可以在子组件中使用 `this.$store` 调用了。
- 当一个组件需要使用多个某 store 的状态属性或 getters ，可以使用 shared helper —— 共享帮手 `mapState`，它会返回一个对象 。

```js
it('helper: mapState (object)', () => {
  const store = new Vuex.Store({
    state: {
      a: 1
    },
    getters: {
      b: () => 2
    }
  })
  const vm = new Vue({
    store,
    computed: mapState({
      // 在 mapState 里面我们既可以调用 store 的 state ，也可以调用 store 的 getters
      a: (state, getters) => {
        return state.a + getters.b
      }
    })
  })
  expect(vm.a).toBe(3)
  store.state.a++
  expect(vm.a).toBe(4)
})
```

那么如何将它与本地的计算属性结合使用呢？一般我们会使用一个工具，将多个对象合而为一，再把这个最终的对象传递给 computed。但是这里我们可以直接使用 es6 的 stage 3 的 object spread operator —— 对象扩展操作符，来超简洁地实现这一功能。

```js
computed: {
  localComputed () {}
  // 将其中的属性与本地的计算属性合并在一起
  ...mapState({
    message: state => state.obj.message
  })
}
```

------
#### Getters 
  有时候我们需要从 store 的状态派生出其他状态，然后对这个状态（的方法）在多个组件中加以利用。通常我们的做法是复制这个方法，或者将它封装为一个公用的方法，然后在需要的时候导入，但是两者其实都不甚理想。Vuex 提供了 getters 属性，用途类似 stores 中的计算属性。
  getters 中的方法接受两个参数，分别为 state 以及 getters（其他 getters），用法如下。

```js
getters: {
  // ...
  doneTodosCount: (state, getters) => {
    return getters.doneTodos.length
  }
}
```

那么我们在其他组件内部使用 getters 也变得十分简单

```js
computed: {
  doneTodosCount () {
    return this.$store.getters.doneTodosCount
  }
}
```

- mapGetters
 可以将 store 的 getters 映射到本地的计算属性中来，除了可以使用数组之外，还可以使用对象起别名。

```js
...mapGetters([
  'doneTodosCount',
  'anotherGetter',
  // ...
])
```

------
#### Mutations
能改变 Vuex store 中的 state 状态的唯一方法是提交 mutation 变更。mutation 和事件很像：都有字符串类型的 type 以及 handler 句柄。我们在 handler 中实际修改 state，state 为每个 mutation 的第一个参数。

```js
const store = new Vuex.Store({
  state: {
    count: 1
  },
  mutations: {
    increment (state) {
      // mutate state
      state.count++
    }
  }
})

// call， 只有在使用 type increment 调用 mutation 时才能称为 handler
store.commit('increment')
```

commit 的第二个可选参数为 payload 有效载荷，可以为普通类型或对象类型等等。
commit 方法还可以通过对象形式调用，这种情况下，这个对象都会被当成 payload 。

```js
store.commit({
  type: 'increment',
  amount: 10
})
```

- little tips
+ 建议使用大写命名 Mutation
将所有大写变量存放在一个文件中，需要的时候引入。使用 es6 的计算属性名新特性来使用常量作为方法名。

```js
// mutation-types.js
export const SOME_MUTATION = 'SOME_MUTATION'
```

```js
// store.js
import Vuex from 'vuex'
import { SOME_MUTATION } from './mutation-types'

const store = new Vuex.Store({
  state: { ... },
  mutations: {
    // we can use the ES2015 computed property name feature
    // to use a constant as the function name
    [SOME_MUTATION] (state) {
      // mutate state
    }
  }
})
```

> es6 计算属性名

```js
// e.g: 使用含有空格的变量作为属性名会报错，此时可以将它存为字符串或者存在中括号包裹的变量中
var lastName = "last name";
var person = {
    "first name": "Nicholas",
    // 中括号包裹的变量
    [lastName]: "Zakas"
};
console.log(person["last name"]); // Zakas
```

+ mutations 必须都是同步的，它的改变必须在调用之后立即执行
因为它是唯一可以修改 state 的，如果它使用了异步方法，将会使我们的 state 变得无法追踪，定位问题也变得是否困难
+ 在组件中 commit mutation 时
可以使用 this.$store.commit() 或者使用 mapMutations 方法，后者可以将组件中的方法映射到 store.commit 调用（需要在根组件注入 store）。

```js
import { mapMutations } from 'vuex'

export default {
  // ...
  methods: {
    // 传入数组
    ...mapMutations([
      'increment' // map this.increment() to this.$store.commit('increment')
    ]),
    // 传入对象，可以使用 alias
    ...mapMutations({
      add: 'increment' // map this.add() to this.$store.commit('increment')
    })
  }
}
```

------
#### Actions
actions 是提交 mutations 的，它可以有任意的异步操作。
actions 的第一个参数是 context，它向外暴露一组与 store 实例相同的方法/属性，所以可以直接调用 context.commit 或者访问 context.state 或者 context.getters 。我们通常使用 es6 的参数解构来简化我们的代码，直接写成 `{ commit }`

```js
actions: {
  increment ({ commit }) {
    commit('increment')
  }
}
```

- 如何触发 Actions？
actions 通过`store.dispatch('actionName')` 触发，其方法体中再触发 mutation，但是 mutations 是可以直接通过 store.commit 触发的，那么为什么不直接使用 store.commit('mutationName') 呢？因为，actions 是可以异步执行的，而 mutations 只可以同步。所以这种 dispatch 调用可以在 action 内执行异步操作，也就是说可以执行异步 mutation。
+ 可以使用 payload 格式或者对象形式触发。二者等价

```js
// dispatch with a payload
store.dispatch('incrementAsync', {
  amount: 10
})

// dispatch with an object
store.dispatch({
  type: 'incrementAsync',
  amount: 10
})
```

+ shopping cart 中的实际应用，既调用了异步 API，又提交了多个 mutation。

```js
actions: {
  checkout ({ commit, state }, payload) {
    // save the items currently in the cart
    const savedCartItems = [...state.cart.added]
    // send out checkout request, and optimistically
    // clear the cart
    commit(types.CHECKOUT_REQUEST)
    // the 异步 shop API accepts a success callback and a failure callback
    shop.buyProducts(
      products,
      // handle success
      () => commit(types.CHECKOUT_SUCCESS),
      // handle failure
      () => commit(types.CHECKOUT_FAILURE, savedCartItems)
    )
  }
}
```

+ 在组件中分发 Actions
可以使用 `this.$store.dispatch()` 或者 `mapActions` 映射组件方法到 `store.dispatch` 中调用（需要注入 root）。同 `mapMutations`
+ Actions 组合，怎么控制 actions 执行呢？
由于 actions 是异步的，因此我们就很难知道一个 action 什么时候完成，以及该怎么把多个 action 组合起来，处理复杂的异步工作流？
好在， `store.dispatch()` 方法返回了我们定义的 action handler 的返回值，所以我们可以直接返回一个 Promise 呀~

```js
actions: {
  actionA ({ commit }) {
    return new Promise((resolve, reject) => {
      setTimeout(() => {
        commit('someMutation')
        resolve()
      }, 1000)
    })
  }
}
```

可以这么用

```js
store.dispatch('actionA').then(() => {
  // ...
})
```

然后在另一个 action 中

```js
actions: {
  // ...
  actionB ({ dispatch, commit }) {
    return dispatch('actionA').then(() => {
      commit('someOtherMutation')
    })
  }
}
```

------
#### Modules
由于 Vuex 使用了单状态树，所以随着我们应用的规模逐渐增大， store 也越来越膨胀。为了应对这个问题，Vuex 允许我们将 store 分成多个 modules。每个 module 有着自己的 state, mutations, actions, getters, 甚至可以有嵌套（ nested ）的 modules。比如说：

```js
const moduleA = {
  state: { ... },
  mutations: { ... },
  actions: { ... },
  getters: { ... }
}

const moduleB = {
  state: { ... },
  mutations: { ... },
  actions: { ... }
}

const store = new Vuex.Store({
  modules: {
    a: moduleA,
    b: moduleB
  }
})

// 注意，调用的时候，多个模块都在 state 对象中，而非 modules 中
store.state.a // -> moduleA's state
store.state.b // -> moduleB's state
```

- modules 中的各种 state ， local or root？
  * mutations 和 getters 中，接受的第一个参数是 modules 的本地 state

```js
const moduleA = {
  state: { count: 0 },
  mutations: {
    increment: (state) {
      // state is the local module state
      state.count++
    }
  },

  getters: {
    doubleCount (state) {
      return state.count * 2
    }
  }
}
```

- 
* 相似地，在 actions 中，`context.state` 为本地 state，而 `context.rootState` 为根 state

```js
const moduleA = {
  // ...
  actions: {
    incrementIfOdd ({ state, commit }) {
      if (state.count % 2 === 1) {
        commit('increment')
      }
    }
  }
}
```

- 
* getters 的第三个参数才是 root state

```js
const moduleA = {
  // ...
  getters: {
    sumWithRootCount (state, getters, rootState) {
      return state.count + rootState.count
    }
  }
}
```

------
#### Strict Mode & Form Handling
严格模式下，如果在 mutation handler 之外修改了 Vuex 的 state，应用就会抛错。比如我们将 Vuex 中的某个数据，用 Vue 的 v-model 绑定到 input 时，一旦感应到 input 改动，就会尝试去直接修改这个数据，严格模式下就会报错。所以建议是绑定 value 值，然后在 input 时调用 action 。

```html
<input :value="message" @input="updateMessage">
```

```js
// ...
computed: {
  ...mapState({
    message: state => state.obj.message
  })
},
methods: {
  updateMessage (e) {
    this.$store.commit('updateMessage', e.target.value)
  }
}
```

mutation 可以这么处理

```js
mutations: {
  updateMessage (state, message) {
    state.obj.message = message
  }
}
```

诚然，这样做是很仔细明了的，但是我们也不能用 v-model 这么好用的方法了，另外一个方法就是继续使用 v-model ，并配套使用 双向计算属性和 setter 。

```js
computed: {
  message: {
    get () {
      return this.$store.state.obj.message
    },
    set (value) {
      // 直接 commit 到 mutation，type 为 updateMessage
      this.$store.commit('updateMessage', value)
    }
  }
}
```

建议部署到开发环境的时候一定一定要关掉严格模式。

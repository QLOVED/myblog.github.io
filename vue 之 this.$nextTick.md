# Vue实现响应式后DOM的变化
## data对象中数据改变是如何追踪的?
vue将遍历data对象中所有的属性，并通过 Object.defineProperty 把这些属性全部转为 getter/setter；但是我们是没有办法看到 getter/setter的，但是在内部它们让 Vue 能够追踪依赖，在属性被访问和修改时通知变更。

每个组件都对应一个 watcher 实例，它会在组件渲染的过程中把“接触”过的数据属性记录为依赖。之后当依赖项的 setter 触发时，会通知 watcher，从而使它关联的组件重新渲染。
 ## Vue是无法检测到data对象属性的添加和删除
 原因：Vue在对初始化组件时会对对象属性执行getter/setter转化，所以属性必须在data对象上存在才能让Vue将它转化为初始化。
 ``` var vm = new Vue({
      data:{
        a:1
      }
    })

    // `vm.a` 是响应式的

    vm.b = 2
    // `vm.b` 是非响应式的
 ```
### 如何动态添加根级别的响应式属性【就是对data添加属性】
`  this.$set(this.someObject,'b',2)`
## 异步更新队列
Vue 在更新 DOM 时是异步执行的。只要侦听到数据变化，Vue 将开启一个队列，并缓冲在同一事件循环中发生的所有数据变更。
如果同一个 watcher 被多次触发，只会被推入到队列中一次。这种在缓冲时去除重复数据对于避免不必要的计算和 DOM 操作是非常重要的。
然后，在下一个的事件循环“tick”中，Vue 刷新队列并执行实际 (已去重的) 工作。
Vue 在内部对异步队列尝试使用原生的 Promise.then、MutationObserver 和 setImmediate，如果执行环境不支持，则会采用 setTimeout(fn, 0) 代替。

例如，当你设置 vm.someData = 'new value'，该组件不会立即重新渲染。
当刷新队列时，组件会在下一个事件循环“tick”中更新。
多数情况我们不需要关心这个过程，但是如果你想基于更新后的 DOM 状态来做点什么，这就可能会有些棘手。虽然 Vue.js 通常鼓励开发人员使用“数据驱动”的方式思考，避免直接接触 DOM，但是有时我们必须要这么做。为了在数据变化之后等待 Vue 完成更新 DOM，可以在数据变化之后立即使用 Vue.nextTick(callback)。这样回调函数将在 DOM 更新完成后被调用。例如：
```
   <div id="example">{{message}}</div>
    var vm = new Vue({
      el: '#example',
      data: {
        message: '123'
      }
    })
    vm.message = 'new message' // 更改数据
    vm.$el.textContent === 'new message' // false
    Vue.nextTick(function () {
      vm.$el.textContent === 'new message' // true
    })
    
```
在组件内使用 vm.$nextTick() 实例方法特别方便，因为它不需要全局 Vue，并且回调函数中的 this 将自动绑定到当前的 Vue 实例上：
```
    Vue.component('example', {
      template: '<span>{{ message }}</span>',
      data: function () {
        return {
          message: '未更新'
        }
      },
      methods: {
        updateMessage: function () {
          this.message = '已更新'
          console.log(this.$el.textContent) // => '未更新'
          this.$nextTick(function () {
            console.log(this.$el.textContent) // => '已更新'
          })
        }
      }
    })
    ```
    因为 $nextTick() 返回一个 Promise 对象，所以你可以使用新的 ES2017 async/await 语法完成相同的事情：
    ```
    methods: {
updateMessage: async function () {

this.message = '已更新'
console.log(this.$el.textContent) // => '未更新'
await this.$nextTick()
console.log(this.$el.textContent) // => '已更新'
}
}
```
    Vue 实现响应式并不是数据发生变化之后 DOM 立即变化，而是按一定的策略进行 DOM 的更新。
$nextTick 是在下次 DOM 更新循环结束之后执行延迟回调，在修改数据之后使用 $nextTick，则可以在回调中获取更新后的 DOM
## 总结
整个过程：
[流程]:https://segmentfault.com/img/remote/1460000020715457
* 1 在new Vue()后， Vue 会调用_init函数进行初始化，也就是init 过程，在 这个过程Data通过Observer转换成了getter/setter的形式，来对数据追踪变化，当被设置的对象被读取的时候会执行getter函数，而在当被赋值的时候会执行setter函数。
* 2 当外界通过Watcher读取数据时，会触发getter从而将Watcher添加到依赖中。
* 3 在修改对象的值的时候，会触发对应的setter，setter通知之前依赖收集得到的 Dep 中的每一个 Watcher，告诉它们自己的值改变了，需要重新渲染视图。这时候这些 Watcher就会开始调用update来更新视图。
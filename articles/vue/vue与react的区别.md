### template vs JSX
template 可以让初学者以html的方式来书写页面，而且JSX则以更加js化的语法来书写页面，当然，vue也支持JSX

### 监听变化
react不知道什么时候应该更新页面，必须要开发者来setState，而vue是通过Object.defineProperty来监听了对象的变化，当对象发生变化的时候，自动触发更新。

vue中用户不需要太过于关心dom是怎么更新的，而react一旦不注意可能会导致一些不需要更新的dom更新。

### 总结
react 中一切都是JS，用JS的方式去编写应用。
 React 设计是改变开发者，提供强大而复杂的机制，开发者按照我的来；Vue 是适应开发者，让开发者怎么爽怎么来。
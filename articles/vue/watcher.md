# watcher

`watcher`的分类：
1. 用于页面更新的`watcher`，每个组件存在一个（`render-watcher`）；
2. 用于`computed`的`watcher`（`computed-watcher`）；
3. 在`watch`中定义的`watcher`（`normal-watcher`）。

执行顺序：`computed-watcher` -> `normal-watcher` -> `render-watcher`**（存疑）**
# mytree

> A Vue.js project
>
> sample.json是样例数据。
>
> 该demo项目是纯前端项目，无后台未配置跨域。
>
> 该项目使用了echarts构造树图，输入的数据格式为类树图数据结构的深度obj数据（属性较少版本，但是结构完整），可根据所用的输入json，调整代码的deepTraversal。
>
> 节点上根据可展开、收起的状态，添加+，-的text标示，为前端生成的data设置一个expandStatus字段标示当前收起展开状态，初始值根据initialTreeDepth设定，在label的formatter中根据expandStatus设置具体的expandStatusIcon。
> 最新发现不需要这么麻烦即可在节点展开收起的时候改变节点的expandStatus，直接为图绑定节点的点击事件，在事件中修改即可，
>
> 原来不是没想到，最开始使用的时候傻乎乎的修改了`this.myTreeChart._chartsViews[0]._data.tree._nodes`中的`expandStatus`的值，这个数据在echarts的渲染逻辑中是不用来管理外部添加的属性的，`this.myTreeChart._chartsViews[0]._data._rawData._data`修改这个字段中的`expandStatus`的值，即可在渲染图的时候直接反应出改变。
>
> 除此之外，还有非常关键的一点，一定要理解这里绑定是一个回调函数，虽然使用`this.myTreeChart._chartsViews[0]._data._rawData._data`成功修改了数据，但是在源代码的逻辑中，这批数据早就绑定过了，用的还是旧数据，所以在修改完成后，我们要使用`resize()`函数二次刷新整张图，以确保最新数据得到正确渲染。
>
> 新方法关键代码段：
>
> ```javascript
> // 修改expandStatus
> this.myTreeChart.on("click", e => {
>   const name = e.data.name;
>   const curNode = this.myTreeChart._chartsViews[0]._data.tree._nodes.find(
>     item => {
>       return item.name === name;
>     }
>   );
>   this.myTreeChart._chartsViews[0]._data._rawData._data.forEach(
>     (item, index) => {
>       if (item.name === curNode.name && item.deep === (curNode.depth - 1)) {
>         item.expandStatus = !item.expandStatus;
>       }
>     }
>   );
>   // 使用resize触发刷新
>   this.myTreeChart.resize();
> });
> ```
> 以下方法为开始笔者探索的时候的蠢办法：
> 在点击节点触发展开，收起时改变被操作节点的expandStatus（通过TreeNode的isExpand属性），具体实现需修改echarts，tree部分treeAction中treeExpandAndCollapse的event的源码，在该事件回调函数底部添加如下代码：
>
> ```javascript
> // 从seriesModel取出重写渲染所需的数据
> let rawData = seriesModel.getData()._rawData._data;
> // 遍历数据找到改变TreeNode的isExpand的节点，修改其渲染数据的expandStatus字段与isExpand一致
> rawData.forEach(
> (item, index) => {
>  if (item.name === node.name && item.deep === (node.depth - 1)) {
>    item.expandStatus = node.isExpand;
>  }
> }
> )
> ```

## Build Setup

``` bash
# install dependencies 先使用install安装依赖
npm install

# serve with hot reload at localhost:8080 然后使用run dev运行项目
npm run dev

# 后续命令为build项目使用
# build for production with minification
npm run build

# build for production and view the bundle analyzer report
npm run build --report
```

For a detailed explanation on how things work, check out the [guide](http://vuejs-templates.github.io/webpack/) and [docs for vue-loader](http://vuejs.github.io/vue-loader).

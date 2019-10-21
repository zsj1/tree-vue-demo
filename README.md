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
> 在点击节点触发展开，收起时改变被操作节点的expandStatus（通过TreeNode的isExpand属性），具体实现需修改echarts，tree部分treeAction中treeExpandAndCollapse的event的源码，在该事件回调函数底部添加如下代码：
>
> ```javascript
> // 从seriesModel取出重写渲染所需的数据
> let rawData = seriesModel.getData()._rawData._data;
> // 遍历数据找到改变TreeNode的isExpand的节点，修改其渲染数据的expandStatus字段与isExpand一致
> rawData.forEach(
>   (item, index) => {
>     if (item.name === node.name && item.deep === (node.depth - 1)) {
>       item.expandStatus = node.isExpand;
>     }
>   }
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

# jsPlumb-example
记录一次基于jsPlumb流程图编辑器的开发过程
### 前言
接到项目需求后，发现没有做过相关项目，盘算着拖拽倒是没有问题，但是控件的连线好像挺复杂，所以先开始了一番搜索，希望有合适的轮子那最好不过了。看了这篇对比文章：[超级好用的流程图js框架](https://blog.csdn.net/AlvinNoending/article/details/52937689)，也看了一个新出的轮子[topology](https://github.com/le5le-com/topology)，选轮子的时候我习惯性去[npm trends](https://www.npmtrends.com/)，找一些类似的轮子横向对比看看各项数据，然后再去对应的仓库找相关文档，做几个demo先试试能满足需求好上手不。<br>
最后选择了[jsPlumb](https://github.com/jsplumb/jsplumb)，选择原因：1.开源。2.官网中的样例基本可以满足我的项目需求。3.文档还算详细，易懂。

### 编辑器结构设计
编辑页面分为左中右三部分，左侧控件部分，中间画板，右侧为流程节点配置编辑区域。左侧控件可以拖拽到中间画板，形成流程中的某个节点，可以进行节点连接。点击或移动节点，右侧编辑页面会打开，可以对节点进行配置编辑。每种类型的节点都对应一个编辑栏的组件，而每一个不同id的节点都有一个配置项数据。

### 控件拖拽入画板

使用[HTML 拖放 API](https://developer.mozilla.org/zh-CN/docs/Web/API/HTML_Drag_and_Drop_API)，主要使用了：

> dragstart：当用户开始拖动一个元素或选中的文本时触发（见[开始拖动操作](https://developer.mozilla.org/zh-CN/docs/Web/API/Document/dragstart_event)）<br>
dragover：当元素或选中的文本被拖到一个可释放目标上时触发（每100毫秒触发一次）<br>
drop：当元素或选中的文本在可释放目标上被释放时触发（见[执行释放](https://developer.mozilla.org/zh-CN/docs/Web/API/Document/drop_event)）

使用示例：
```html
<div class="board">
    <ul class="menus">
        <li class="menu"
            v-for="menu in menus"
            :key="menu.type"
            draggable
            @dragstart="dragstart($event, menu)"
        >
            <i :class="`iconfont ${menu.iconName}`"></i>
        </li>
    </ul>
    <main class="drag-content" @dragover.prevent @drop.prevent="drop">
        <div v-for="item in flowList" :key="item.id" class="drag-menu" :style="item.position">
            <i :class="`iconfont ${item.iconName}`"></i>
        </div>
    </main>
    <aside>
        <component :is="'tpl' + menuType" :info="currentMenu.config"></component>
    </aside>
  </div>
```
```js
methods: {
    dragstart(event, menu) {
        event.dataTransfer.setData('text/plain', JSON.stringify(menu))
    },
    drop(event) {
        let data = event.dataTransfer.getData('text/plain')
        let dom = document.querySelector('.drag-content')
        let left = event.x - dom.offsetLeft + dom.scrollLeft
        let top = event.y - dom.offsetTop + dom.scrollTop
        data = JSON.parse(data)
        left = left - left % 4 + 'px'   // 网格grid: [4, 4]， 便于直线对齐（优化）
        top = top - top % 4 + 'px'
        data.id = uuidv1()
        data.position = { left, top }
        this.flowList.push(data)
    }
  }
```
1. 使用drag event 的 dataTransfer属性保存自定义的控件数据，进行传递。另外也可以定义一个currentMenu保存数据，但这需要重置。<br>
2. 画布中的元素是相对画布定位的，所以在drog事件中需要对坐标进行处理。<br>
3. 在dragover事件中event.preventDefault()很关键。

### 节点拖拽连接
流程图画板的操作，主要是基于jsPlumb。<br>
官方文档 <http://jsplumb.github.io/jsplumb/home.html><br>

#### 1.jsPlumb默认是在浏览器的窗口上注册的，为整个页面提供一个静态实例。可以使用getInstance方法来实例化jsPlumb的独立实例。并设置默认容器，限制在容器内拖拽。
```js
mounted() {
    this.instance = jsPlumb.getInstance()
    this.instance.setContainer('containerId')
    this.instance.ready(() => {
      // 绘制，如连接锚点
    })
}
```
#### 2.设置节点可拖拽，instance.draggable
 - arrays<br>
`jsPlumbInstance.draggable(["elementOne", "elementTwo"]);`
 - jQuery selectors<br>
`jsPlumbInstance.draggable($(".someClass"));`
 - NodeLists<br>
 `var els = document.querySelectorAll(".someClass")`<br>
 `jsPlumbInstance.draggable(els)`

使用示例：
```js
this.$nextTick(() => {
    var els = document.querySelectorAll('.drag-menu')
    this.instance.draggable(els, {
        containment: true,  // 设置父元素为默认容器， 与上面setContainer作用一样。
        grid: [4, 4]   // 要将元素约束到网格中，移动的时候不再平滑，但是便于节点对齐。
    })
})
```
#### 3.设置锚点，instance.addEndpoint
jsPlumb源码中显示addEndpoint方法接受三个参数，节点元素的id，后面的是配置参数会合并。
```javascript
// --------------------------- jsPlumbInstance public API  -----------------------------
        this.addEndpoint = function (el, params, referenceParams) {
            referenceParams = referenceParams || {};
            var p = jsPlumb.extend({}, referenceParams);
            jsPlumb.extend(p, params);
            p.endpoint = p.endpoint || _currentInstance.Defaults.Endpoint;
            p.paintStyle = p.paintStyle || _currentInstance.Defaults.EndpointStyle;
			...
```
使用示例：
```javascript
let uuid = uuidv1()
this.instance.addEndpoint(elementId, {
    anchor: [ 0.5, 0, 0, -1, 0, 0, 'anchor-name'],
    uuid,
    isSource: true,
    isTarget: true,
    paintStyle: { fill: '#D8DDE6', radius: 5 },
    connector: 'Flowchart',
    connectorStyle: { stroke: '#D8DDE6', strokeWidth: 2 },
    connectorOverlays: [
        [
            'Arrow',
            {   
                id: uuid  // 这里是为了监听到连接事件后，传递锚点uuid
                location: 0.5,
                width: 8,
                height: 5,
                paintStyle: { stroke: '#D8DDE6', fill: '#D8DDE6' }
            }
        ],
        ['Label', { label: 'xxxx', location: 0.25 }]
    ]
});
```
重点参数说明：<br>
1. anchor：锚点位置有四种类型
    - Static：静态，会固定到元素上的某个点，不会移动
    - Dynamic：动态，是静态锚的集合，就是jsPlumb每次连接时选择最合适的锚
    - Perimeter anchors：周边锚，动态锚的应用
    - Continuous anchors：连续锚
    
    这里着重解释下使用基于数组的形式来定义锚点位置：[x,y,dx,dy,offsetX,offsetY]<br>
    - x表示锚点在横轴上的距离，y表示锚点在纵轴上的距离，这两个值可以从0到1来设置，0.5为center
    - dx表示锚点向横轴射出线，dy表示锚点向纵轴射出线，有0，-1，1三个值来设置。0为不放射线（控制从锚点射出的连接线的初始方向）
    - offsetX表示锚点横轴偏移量x（px），offsetY表示锚点纵轴偏移量y（px）
2. connectorOverlays：连接线样式设置
    - Arrow--一个可配置的箭头，沿着连接器的某个点涂上。你可以控制箭头的长度和宽度，"折返 "点--尾部折返的点，以及方向（允许的值是1和-1；1是默认值，意味着指向连接的方向）
    - Label - 可配置的标签，沿着连接器的某个点涂上标签。
    - PlainArrow - 箭头形状为三角形的箭头，没有反折。
    - Diamond - 菱形。
    - Custom - 允许你自己创建 Overlay - 你的 Overlay 可以是你喜欢的任何 DOM 元素。
    - PlainArrow和Diamond实际上只是一般Arrow覆盖层的配置实例）。
    
    其中可以对同一个节点有多个锚点，各个锚点分别设置不同的label，区分不同的分支。这是两个锚点直接的桥梁，关联节点关系的时候使用。而分支名的字典表和其代表的含义，可以在右侧编辑栏中配置。
3. connector: 连接线svg形状
    - Straight: 直线
    - Bezier: 贝塞尔曲线
    - Flowchart: 具有90度转折点的流程线
    - StateMachine: 状态机
#### 3.连接锚点，instance.connect
- 可以手动拖动锚点直接连接到另外一个节点的锚点，其中要区分（isSource和isTarget），添加锚点的时候设置了此锚点是出发点还是终点。
- 可以使用instance.connect连接两个锚点。

    我在创建锚点的时候，在配置参数里面添加uuid，然后通过uuid连接
    ```js
    this.instance.addEndpoint("elId1", { uuid:"abcdefg" }, endpointOptions )
    this.instance.addEndpoint("elId2", { uuid:"hijklmn" }, endpointOptions )
    this.instance.connect({uuids:["abcdefg", "hijklmn"]})
    ```
#### 3.删除锚点 
- 删除单一锚点
    ```js
    var ep = this.instance.addEndpoint(elId, { ... })
    ...
    this.instance.deleteEndpoint(ep)
    ```
- 删除所有锚点<br>
  同时也删除每个连接，这个方法和jsPlumb.reset很相似，只是这个方法不删除已注册的事件处理程序
  ```js
    this.instance.deleteEveryEndpoint()
    ```
#### 4.建立节点关系 
- 监听锚点连接事件，其中targetId和sourceId我们可以获取节点id，另外connectorOverlays[0][1].id就是对于锚点的id，这里记录锚点id，维护一份`this.connectIds`是为了再次编辑的时候，能通过`instance.connect({uuids})`直接连接节点。
    ```js
    数据格式：this.flowList = [{
        type: '',     // 节点类型
        id: '',       // uuid
        parents: [],  // 父节点id
        children: [], // 子节点id
        points: [],   // 自身锚点信息
        config: {},   // 节点配置信息
        branch: {}    // 分支信息，就是连线上的label信息
    }]
    this.instance.bind('connection', info => {
        let source, target
        for (let item of this.flowList) {
            if (source && target) break
            if (!source && item.id === info.sourceId) {
                source = item
            } else if (!target && item.id === info.targetId) {
                target = item
            }
        }
        // source和target一定能find到
        if (!source.children.includes(info.targetId)) {
            source.children.push(info.targetId)
        }
        if (!target.parents.includes(info.sourceId)) {
            target.parents.push(info.sourceId)
        }
        if (info.sourceEndpoint.connectorOverlays[1]) {
            target.branch = info.sourceEndpoint.connectorOverlays[1][1].label
            // 我将分支信息带在子节点上
        }
        let sid = info.sourceEndpoint.connectorOverlays[0][1].id
        let tid = info.targetEndpoint.connectorOverlays[0][1].id
        // 点击某个锚点，未断开连接就直接连接新的锚点，需要删除之前的链接
        this.connectIds = this.connectIds.filter(el => el[0] !== sid && el[1] !== tid)
        this.connectIds.push([sid, tid])
        // 这些connectIds是预览流程图的时候直接连接锚点使用
    })
    ```
- 监听断开连接事件，将存储的锚点关系列表进行数据更新。
    ```js
    this.instance.bind('connectionDetached', info => {
        // 同上bind('connection')，find到source、target
        source.children = source.children.filter(id => id !== info.targetId)
        target.parents = target.parents.filter(id => id !== info.sourceId)
        let sid = info.sourceEndpoint.connectorOverlays[0][1].id
        let tid = info.targetEndpoint.connectorOverlays[0][1].id
        this.connectIds = this.connectIds.filter(el => !_.isEqual(el, [sid, tid]))
    })
    ```
### 总结
整个编辑画板中的节点元素、锚点、连线都是兄弟元素，都是相对于父节点定位，在节点附近自定义添加其他元素也比较容易，其中连线是采用svg绘制，其中正交连线我觉得是有一定难度的。项目中的元素分支后续可以优化，目前是在确定锚点的时候确定了此分支的名称，当从锚点拽出分支线的时候，其中的label标签才会告诉用户这是什么分支。这一步可以提前，在生成锚点的时候自动生成连接线并展示出分支名。<br>
这是近期项目的自我记录，不方便放UI图和整体源码，如果有做流程图的同行需要交流可私聊。文章最后我会贴上学习jsPlumb的相关文章 ：

> 官方文档: <http://jsplumb.github.io/jsplumb/home.html><br>
前辈总结的中文教程: <https://wdd.js.org/jsplumb-chinese-tutorial/#/> <br>
jsPlumb.jsAPI阅读笔记: <https://www.cnblogs.com/leomyili/p/6346526.html> <br>
流程图-正交连线的算法的一种简单实现: <https://juejin.im/post/5b73829ee51d4566205fe7f0>

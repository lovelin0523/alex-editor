# 基于原生 JS 的 L1 级别富文本编辑器 alex-editor

### 特性

-   基于数据驱动：每次操作后会先更新编辑器对象内部保存的一组特殊的数据结构，然后通过数据来重新渲染富文本编辑器的内容
-   没有使用 document.execCommand 语法，内部通过浏览器的 selection/range 对象来操作光标，配合数据驱动，实现任何你想要的操作
-   上手简单，使用方便

### 快速上手

```html
<div id="editor" style="width:500px;height:500px;"></div>
```

```javascript
import AlexEditor from 'alex-editor'
const el = document.body.querySlector('#editor')
const editor = new AlexEditor(el, {
	autofocus: true,
	value: '<p>hello,我是一个编辑器</p>'
})
```

此时，一个基本的富文本编辑器已经创建好了，但是需要注意的是，这个富文本编辑器仅仅是一个可输入的区域，没有任何其他额外的功能

如果你需要使用复杂的功能，或者需要做一些自定义的操作，你需要引入 AlexElement 对象，同时搭配 AlexRange 实例来实现

```javascript
//引入AlexElement对象
import { AlexEditor, AlexElement } from 'alex-editor'
const el = document.body.querySlector('#editor')
const editor = new AlexEditor(el, {
	autofocus: true,
	value: '<p>hello,我是一个编辑器</p>'
})
//获取AlexRange实例
const range = editor.range
```

如下代码实现了将一段文本插入到文档尾部：

```javascript
//创建一个文本元素
const ele = new AlexElement('text', null, null, null, '我是插入的一段文本')
//将光标设置到编辑器内容尾部
editor.range.collapseToEnd()
//将创建的文本添加到光标所在元素的后面
editor.addElementAfter(ele, editor.range.focus.element)
//将range的起点和终点移动到创建的文本后面
editor.range.anchor.moveToEnd(ele)
editor.range.focus.moveToEnd(ele)
//渲染
editor.formatElementStack()
editor.domRender()
editor.range.setCursor()
```

> 自定义操作中，最后都需要使用 editor.formatElementStack、editor.domRender 和 editor.range.setCursor()，这三部分按顺序使用，缺一不可。主要作用是格式化编辑器元素数组、渲染编辑器 dom 内容，设置真实光标位置

### 创建 editor 实例的第二个构造参数 options 是一个对象，具体包含以下属性：

| 属性            | 类型     | 说明                                                                                                                                                                                        | 可取值     | 默认值           |
| --------------- | -------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------- | ---------------- |
| value           | string   | 编辑器的 html 内容，可以实时获取到编辑器的内容                                                                                                                                              | -          | "\<p>\<br>\</p>" |
| disabled        | boolean  | 是否禁用编辑器                                                                                                                                                                              | true/false | false            |
| renderRules     | function | 自定义编辑器格式化规则，回调参数为 element，表示当前要渲染的 AlexElement 实例，你可以针对该实例或者其子孙元素进行操作，并将该元素返回（不能修改父子元素关系，要么直接从父组件中删除子元素） | -          | -                |
| autofocus       | boolean  | 是否自动获取焦点                                                                                                                                                                            | true/false | false            |
| onChange        | function | 编辑器值更新时触发，回调参数为 newValue 和 oldValue，同时 this 指向编辑器实例                                                                                                               | -          | -                |
| htmlPaste       | boolean  | 粘贴时是否携带样式                                                                                                                                                                          | true/false | false            |
| handlePasteFile | function | 自定义文件粘贴的处理函数，回调参数为文件数组 files，如果不设置，编辑器只会以把 base64 字符串形式加载图片和视频，其他文件直接忽略                                                            | -          | -                |

### 编辑器内部元素规范

> 即 editor.formatElementStack 函数执行效果

-   子元素中换行符 \<br> 和其他元素不可同时存在（如果同时存在换行符会被移除）
-   根元素必须全都是 block 类型的元素（如果根元素存在其他类型的元素，会进行一次强制转换，即调用 convertToBlock 方法进行转换）
-   同级元素中 block 元素和其他元素不能同时存在，否则会把其他元素强转为 block 元素
-   执行格式化时会把空元素移除（text 元素内容为空视为空元素、block 和 inline 没有子元素视为空元素）
-   AlexPoint 对象的 element 属性只会是 closed 和 text 元素
-   删除操作中，如果 block 元素的 children 为空了，则添加一个换行符 \<br>，在删除换行符后才会清除此元素
-   如果 AlexPoint 对象的 element 是 closed 元素，那么 offset 要么是 0 要么是 1
-   操作编辑器只能通过 AlexElement 对象和 AlexEditor 对象，不支持 node 和 html 直接插入

> 以上的规则是编辑器内部渲染的一个固定的逻辑，无法去改变。但是我们提供了一个 renderRulers 函数，来使得我们可以对元素做一些额外的操作

```javascript
const editor = new AlexEditor(el, {
	renderRules: function (element) {
		//将span设置为block
		if (element.parsedom == 'span') {
			element.type = 'block'
			element.styles = {
				display: 'block'
			}
		}
		return element
	}
})
```

### AlexEditor 编辑器

作为该编辑器组件的最顶级的核心类，其功能强大，提供了丰富的语法：

-   `editor.$el` ：编辑器所在的 dom 元素
-   `editor.value` ：当前编辑器的内容
-   `editor.range` ：editor 内部创建的 AlexRange 实例，通过该属性来操控 anchor、focus 和设置光标。请勿修改此属性修改
-   `editor.stack` ：存放编辑器内所有的 AlexElement 元素的数组
-   `editor.history` ：editor 内部创建的 AlexHistory 实例，通过该属性来操控历史的记录
-   `editor.setRecentlyPoint(point)` : 将指定焦点的元素设置为前后最近的 closed 或者 text 元素
-   `editor.getPreviousElement(ele)` ：获取 ele 元素前一个兄弟元素，如果没有则返回 null
-   `editor.getNextElement(ele)` ：获取 ele 元素后一个兄弟元素，如果没有则返回 null
-   `editor.addElementTo(childEle, parentEle, index)` ：将指定元素添加到父元素的子元素数组中
-   `editor.addElementBefore(newEle, targetEle)` ：将指定元素添加到另一个元素前面
-   `editor.addElementAfter(newEle, targetEle)` ：将指定元素添加到另一个元素后面
-   `editor.mergeBlockElement(ele)` ：将指定的块元素与其前一个块元素进行合并
-   `editor.getElementByKey(key)` ：根据 key 查询元素
-   `editor.parseNode(node)` ：将 node 节点转为 AlexElement 元素
-   `editor.parseHtml(html)` ：将 html 文本内容转为 AlexElement 元素，返回一个元素数组
-   `editor.getPreviousElementOfPoint(point)` ：根据指定焦点向前查询可以设置焦点的最近的元素
-   `editor.getNextElementOfPoint(point)` ：根据指定焦点向后查询可以设置焦点的最近的元素
-   `editor.getElementsByRange()` ：获取 anchor 和 focus 两个焦点之间的元素（如果焦点在文本中间，还会分割文本元素）
-   `editor.formatElement(ele)` ：对传入的 AlexElement 实例进行格式化，返回格式化后的元素
-   `editor.formatElementStack()` ：对 editor.stack 进行内部的格式化规范校验处理
-   `editor.domRender(unPushHistory)` ：渲染编辑器 dom 内容，该方法会触发 value 的更新，如果 unPushHistory 为 true，则本次操作不会添加到历史记录中去，除了做“撤销”和“重做”功能时一般情况下不建议设置此参数
-   `editor.setDisabled()` ：设置编辑器禁用，此时不可编辑
-   `editor.setEnabled()` ：设置编辑器启用，此时可以编辑
-   `editor.delete()` ：根据光标执行删除操作
-   `editor.insertText(data)` ：根据光标位置向编辑器内插入文本
-   `editor.insertParagraph()` ：在光标处换行
-   `editor.insertElement(ele)` ：根据光标位置插入指定的 ele，可用于在生成元素后向编辑器内插入
-   `editor.collapseToStart(element)` ：将光标移动到文档头部，如果 element 指定了元素，则移动到该元素头部
-   `editor.collapseToEnd(element)` ：将光标移动到文档尾部，如果 element 指定了元素，则移动到该元素尾部
-   `editor.setStyle(styleObject)` ：根据光标设定指定的样式，参数是一个对象，key 表示 css 样式名称，value 表示值
-   `editor.destroy()` ：销毁编辑器，主要是设置编辑器不可编辑，同时移除编辑相关的事件。当编辑器对应的元素从页面中移除前，应当调用一次该方法进行事件解绑处理

### AlexElement：元素

AlexElement 是 `alex-editor` 定义的一种特殊的数据结构，编辑器初始化时将 html 内容转为 AlexElement 数组，并挂载在编辑器实例上（ `editor.stack` ），后续的任意操作都将通过修改该组数据结构来更新编辑器内容。
其构造函数包含以下几个参数：

-   type：元素类型，可取值"text"（文本元素）、"closed"（自闭合元素）、"inline"（行内元素）、"block"（块元素）
-   parsedom：对应的需要渲染的真实节点名称（例："p"），如果是文本元素，此项为 null
-   marks：元素标记集合，对应需要渲染的真实节点的属性（不包括 style 和事件），如果是文本元素，此项为 null
-   styles：元素样式集合，对应需要渲染的真实节点的样式，如果是文本元素，此项为 null
-   textContent：文本内容，非文本元素下此值为 null

```javascript
//创建一个图片元素
const imageElement = new AlexElement('closed', 'img', { src: '#' }, { width: '300px' }, null, null)

//创建一个文本元素
const textElement = new AlexElement('text', null, null, null, null, '我是一个文本')
```

> 元素创建后并添加到堆栈中，拥有唯一的 key 属性，并且可以通过 parent 属性访问父元素，通过 children 属性来访问子元素

AlexElement 提供以下几种语法来方便我们的操作：

-   `el.isText()` ：el 是否文本元素
-   `el.isBlock()` ：el 是否块元素
-   `el.isInline()` ：el 是否行内元素
-   `el.isClosed()` ：el 是否自闭合元素
-   `el.isBreak()` ：el 是否是换行符
-   `el.isEmpty()` ：el 是否是空元素。文本没有值，行内和块元素没有子元素或者子元素都是 null 的话，都是空元素
-   `el.isRoot()` ：el 是否是根元素，即 AlexElement.elementStack 数组中的元素
-   `el.isContains(element)` ：el 是否包含 element。如果两个元素相等也认为是包含关系
-   `el.isEqual(element)` ：el 是否与 element 相等，即二者是否同一个元素
-   `el.hasContains(element)` ：el 与 element 是否拥有包含关系。即 el 包含 element 或者 element 包含 el 都视为拥有包含关系
-   `el.hasMarks()` ：el 是否含有标记
-   `el.hasStyles()` ：el 是否含有样式
-   `el.hasChildren()` ：el 是否有子元素
-   `el.clone(deep)` ：将 el 元素进行克隆，返回一个新的元素，deep 为 true 表示深度克隆，即克隆子孙元素，默认为 true
-   `el.convertToBlock()` ：将非 block 类型的元素转为 block 元素
-   `el.setEmpty()` ：将一个非空元素设置为空元素（如果你希望在编辑器内部进行格式化的时候删除此元素，可以使用此方法设为空元素，因为空元素会在格式化时删除）
-   `AlexElement.paragraph` ：定义段落元素，默认是"p"
-   `AlexElement.isElement(val)` ：判断 val 是否 AlexElement 对象
-   `AlexElement.flatElements(elements)` ：将 elements 元素数组转为扁平化元素数组

> 自行创建的 AlexElement 元素实例，向编辑器内插入需要添加到某元素的 children 里，并且该元素的 parent 设为某元素。你可以选择 editor.addElementTo、editor.addElementBefore 和 editor.addElementAfter 来插入新的元素，此时不需要你自己设置 children 和 parent

> 自行创建的 AlexElement 实例可能不符合编辑器内部的规范，在插入编辑器并渲染后可能和我们预想的效果不太一样，甚至会出现 bug。因此在创建元素后，必须使用 editor.formatElement(element)方法来进行格式化，该方法会返回格式化后的元素（通过 parseNode 和 parseHtml 生成的 AlexElement 元素是已经格式化后的了，无需再格式化）

### AlexPoint：焦点对象

AlexPoint 表示当前操作的光标焦点，由编辑器内部创建其实例，包含 element 和 offset 两个属性

-   elemet: 即 AlexElement 实例，即该焦点所在的元素，只能是自闭合元素或者文本元素
-   offset：即焦点在元素上的偏移值，如果是自闭合元素，只能是 0 和 1

AlexPoint 提供以下几种语法来方便我们的操作：

-   `point.isEqual(target)` ：point 是否和 target 相等，即两个焦点是否同一个
-   `point.moveToEnd(element)` ：将焦点移动到 element 之后，如果 element 不是自闭合元素和文本元素，会查找其最后一个子元素，以此类推，直至获取到自闭合元素或者文本元素
-   `point.moveToStart(element)` ：将焦点移动到 element 之前，如果 element 不是自闭合元素和文本元素，会查找其第一个子元素，以此类推，直至获取到自闭合元素或者文本元素
-   `point.getBlock()` ：获取该焦点所在的块元素
-   `point.getInline()` ：获取该焦点所在的行内元素
-   `AlexPoint.isPoint(val)` ：判断 val 是否 AlexPoint 对象

### AlexRange：光标范围对象

AlexPoint 是基于浏览器原生的 Selection 和 Range 对象进行封装，表示光标的相关操作，由编辑器内部创建其实例并不断更新，包含 anchor 和 focus 两个属性：

-   anchor：起点，是一个 AlexPoint 实例
-   focus：终点，是一个 AlexPoint 实例

> 如果 anchor 和 focus 相等，则表示光标在某一处，没有选择区域；如果 anchor.element 和 foucs.element 相等，则表示光标在一个元素上

AlexRange 提供下面的方法来方便我们的操作：

-   `range.setCursor()` ：根据 anchor 和 focus 来设置编辑器真实光标位置

> 自定义操作中如果使用的是 editor 提供的语法，如 insertText，insertElement，delete 等等，会更新 range 的光标焦点位置。如果你是自己操作，不依赖于这些语法，你需要手动去更新 range 的 anchor 和 focus。

### AlexHistory：历史记录对象

AlexHistory 是 editor 内部封装的一个用于读取和存入历史记录的对象，可以用来实现撤销和重做的功能，包含两个属性：

-   records：一个数组，数组内的元素包含 stack 和 range 两个属性，stack 表示记录的元素数组，range 表示记录的光标信息
-   current：记录当前编辑器展示的 stack 和 range 在 records 中的序列

AlexHistory 通过以下方法来读取和操作历史记录：

-   `history.get(type)` ：type 为-1 时表示读取上一个历史记录，type 为 1 时表示读取下一个历史记录，返回一个包含 stack 和 range 的对象
-   `history.push(stack,range)` ：将 stack 和 range 保存在历史记录中

> editor 在每次执行 domRender 时，当 unPushHistory 参数为 false 时，都会自动把当前编辑器的 stack 和 range 保存到 history 中去

> 自定义撤销和重做功能时，你需要通过 get 方法读取 stack 和 range，然后赋值给编辑器的 stack 和 range，然后执行 editor.domRender(true)并通过 range.setCursor()设置光标

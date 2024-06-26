https://developer.mozilla.org/zh-CN/docs/Web/API/MouseEvent

**clientX** 是只读属性，它提供事件发生时的应用客户端区域的水平坐标 (与页面坐标不同)。例如，不论页面是否有水平滚动，当你点击客户端区域的左上角时，鼠标事件的 clientX 值都将为 0。最初这个属性被定义为长整型（long integer），如今 CSSOM 视图模块将其重新定义为双精度浮点数（double float）。

**offsetX** 规定了事件对象与目标节点的内填充边（padding edge）在 X 轴方向上的偏移量。

**pageX** 是相对于整个文档的 x（水平）坐标以像素为单位的只读属性。
这个属性将基于文档的边缘，考虑任何页面的水平方向上的滚动。举个例子，如果页面向右滚动 200px 并出现了滚动条，这部分在窗口之外，然后鼠标点击距离窗口左边 100px 的位置，pageX 所返回的值将是 300。
起初这个属性被定义为长整型。CSSOM 视图模块将它重新定位为双浮点数类型。

**screenX** 是鼠标在全局（屏幕）中的水平坐标（偏移量）。
https://stackoverflow.com/questions/21064101/understanding-offsetwidth-clientwidth-scrollwidth-and-height-respectively

**offsetWidth, offsetHeight**: 这些属性表示包括所有边框在内的视觉框的大小。如果元素具有display: block，可以计算为宽度/高度加上内边距和边框。
**clientWidth, clientHeight**: 这些属性表示内容的可视部分，不包括边框或滚动条，但包括内边距。不能直接从CSS计算，因为它取决于系统的滚动条大小。
**scrollWidth, scrollHeight**: 这些属性表示盒子内容的总大小，包括当前隐藏在滚动区域之外的部分。不能直接从CSS计算，因为它们依赖于内容的大小。
[参考](https://jsfiddle.net/y8Y32/25/)

请注意，这些属性都是只读的，并且返回的是整数，可能会受到舍入误差的影响。在处理布局和滚动时，理解这些属性的区别至关重要。
如果需要精确值，使用**Element.getBoundingClientRect()**:
返回一个 DOMRect 对象，是包含整个元素的最小矩形（包括 padding 和 border-width）。该对象使用 left、top、right、bottom、x、y、width 和 height 这几个以像素为单位的只读属性描述整个矩形的位置和大小。除了 width 和 height 以外的属性是相对于视图窗口的左上角来计算的。
---
template: main.html
tags:
  - Golang
  - Goquery
---

# Goquery 常用使用姿势

| 选择器               | 示例                                              | 注释                                                |     |
| -------------------- | ------------------------------------------------- | --------------------------------------------------- | --- |
| 标签选择器           | ("#myelement")                                    | 选择 id                                             |     |
| 标签选择器           | ("div")                                           | 选择\<tag>标签                                      |     |
| 标签选择器           | (".MyClass")                                     | 选择 class                                          |     |
| 标签选择器           | ("\*")                                            | 选择所有标签                                        |     |
| 标签选择器           | ("#myelement, div,. MyClass")                     | 选择多个标签                                        |     |
|                      |                                                   |                                                     |     |
| 级联选择器           | ("form input")                                    | 选择`input`标签的内容                               |     |
| 级联选择器           | ("#main > \*")                                    | 选择 id 的所有子标签                                |     |
| 级联选择器           | ("label + input")                                 | 选择`input`中指定`label`后的所有标签                |     |
| 级联选择器           | ("#prev ~ div")                                   | 兄弟选择器，选择`div`的同级标签，共有一个父标签 id  |
|                      |                                                   |                                                     |     |
| 基本过滤选择器       | ("tr: first")                                     | 选择所有`tr`标签的首元素                            |     |
| 基本过滤选择器       | ("tr: last")                                      | 选择所有`tr`标签的尾元素                            |     |
| 基本过滤选择器       | ("input:not(:checked) + span")                    | 选择`input`标签中所有 checked 标签                  |     |
| 基本过滤选择器       | ("tr: even")                                      | 选择`tr`标签中 0, 2, 4...元素                       |     |
| 基本过滤选择器       | ("tr: odd")                                       | 选择`tr`标签中 1, 2, 5...元素                       |     |
| 基本过滤选择器       | ("Td: EQ (2)")                                    | 选择`TD`标签中序号为 2 的元素                       |     |
| 基本过滤选择器       | ("Td: TQ (2)")                                    | 选择`TD`标签中序号大于 2 的元素                     |     |
| 基本过滤选择器       | ("Td: LL (2)")                                    | 选择`TD`标签中序号小于 2 的元素                     |     |
| 基本过滤选择器       | (":header")                                       | 选择`header`                                        |     |
| 基本过滤选择器       | ("div:animated")                                  | 选择``                                              |     |
|                      |                                                   |                                                     |     |
| 内容过滤选择器       | ("div: contains ('john')")                        | 选择包含文本'john'的`div`标签                       |     |
| 内容过滤选择器       | ("Td: empty")                                     | 选择所有空的`TD`标签，或者不包含 text               |     |
| 内容过滤选择器       | ("div: has (P)")                                  | 选择包含`p`标签的`div`标签                          |     |
| 内容过滤选择器       | ("Td: parent")                                    | 选择`TD`标签的子标签                                |     |
|                      |                                                   |                                                     |     |
| 视觉过滤选择器       | ("div: hidden")                                   | 选择所有隐藏标签                                    |     |
| 视觉过滤选择器       | ("div: visible")                                  | 选择所有可视标签                                    |     |
|                      |                                                   |                                                     |     |
| 属性过滤选择器       | ("div [ID]")                                      | 选择所有包含指定`id`的`div`标签                     |     |
| 属性过滤选择器       | ("input [name = 'newsletter'])                    | 选择标签中`name`等于`newsletter`的`input`所有标签   |     |
| 属性过滤选择器       | ("input [name != 'newsletter'])                   | 选择标签中`name`不等于`newsletter`的`input`所有标签 |     |
| 属性过滤选择器       | ("input [name ^= 'news'])                         | 选择标签中`name`开头为`news`的`input`所有标签       |     |
| 属性过滤选择器       | ("input [name $= 'news'])                         | 选择标签中`name`结尾为`news`的`input`所有标签       |     |
| 属性过滤选择器       | ("input [name *= 'news'])                         | 选择标签中`name`包含`news`的`input`所有标签         |     |
| 属性过滤选择器       | ("input [ID] [name $='news'])                     | 联合选择                                            |     |
|                      |                                                   |                                                     |     |
| 子元素过滤选择器     | ("ul li:nth-child(2)")                            |                                                     |     |
| 子元素过滤选择器     | ("ul li:nth-child(odd)")                          |                                                     |     |
| 子元素过滤选择器     | ("ul li:nth-child(3n + 1)")                       |                                                     |     |
| 子元素过滤选择器     | ("div span: first child")                         | 选择所有`div`中，`span`的第一个子节点               |     |
| 子元素过滤选择器     | ("div span: last child")                          | 选择所有`div`中，`span`第一个尾节点                 |     |
| 子元素过滤选择器     | ("div span: only child")                          | 选择所有`div`中，`span`中唯一的一个节点             |     |
|                      |                                                   |                                                     |     |
| 表单选择器           | (":input")                                        | 选择所有`input`元素                                 |     |
| 表单选择器           | (":text")                                         | 选择所有`text`元素                                  |     |
| 表单选择器           | (":password")                                     | 选择所有`password`元素                              |     |
| 表单选择器           | (":Radio")                                        | 选择所有`Radio`元素                                 |     |
| 表单选择器           | (":checkbox")                                     | 选择所有`checkbox`元素                              |     |
| 表单选择器           | (":Submit")                                       | 选择所有`Submit`元素                                |     |
| 表单选择器           | (":image")                                        | 选择所有`image`元素                                 |     |
| 表单选择器           | (":reset")                                        | 选择所有`reset`元素                                 |     |
| 表单选择器           | (":button")                                       | 选择所有`button`元素                                |     |
| 表单选择器           | (":file")                                         | 选择所有`file`元素                                  |     |
| 表单选择器           | (":hidden")                                       | 选择所有`hidden`类型元素                            |     |
|                      |                                                   |                                                     |     |
| 表单元素过滤器选择器 | (": enabled")                                     | 选择所有可操作元素                                  |     |
| 表单元素过滤器选择器 | (": disabled")                                    | 选择所有不可操作元素                                |     |
| 表单元素过滤器选择器 | (": checked")                                     | 选择所有 checked 元素                               |     |
| 表单元素过滤器选择器 | ("select option: selected")                       | 联合选择                                            |     |
|                      |                                                   |                                                     |     |
|                      | ("input[@ name =S_03_22]").parent().prev().text() |                                                     |     |
|                      | ("input[@ name ^= 'S_']").not("[@ name $='_R']")  |                                                     |     |
|                      | ("input@ name =radio_01").val()                   |                                                     |     |
|                      | ("a, B")                                          | 选择`a`子元素，包含间接子元素                       |     |
|                      | ("a > B")                                         | 选择`a`直属子元素                                   |     |
|                      | ("a + B")                                         | 选择`a`后面的兄弟节点，包括间接子节点               |     |
|                      | ("a ~ B")                                         | 选择`a`后面的兄弟节点，不包括间接子节点             |     |

# 虚拟渲染树行表格实现原理

### 虚拟树渲染逻辑

@startuml start :基于树行结构数据构建一颗 \*\*Fiber\*\* 树; repeat :遍历 Fiber 树，找到标记展开的节点; :将节点添加到渲染列表（平铺结构）; :传入 react-window 进行虚拟渲染; note right 渲染数据节点为原始 Fiber 节点， 保留 sibling、return、child 指针 end note :用户交互：展开/收起; backward: 基于获取的 Fiber 节点，遍历整颗 \*\*Fiber\*\* 树; repeat while(获取当前交互 \*\*Fiber\*\* 节点) stop @enduml

#### Filber 数据结构

### ![filber.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/jP2lRX1YJkPJq8g5/img/14db9b99-4bc8-46b6-b459-ea7c4d371c04.png)

### 虚拟列表渲染逻辑

@startuml start if (Graphviz installed?) then (yes)   :process all diagrams; else (no)   :process only   \_\_sequence\_\_ and \_\_activity\_\_ diagrams; endif stop @enduml   

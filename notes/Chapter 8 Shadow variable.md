# Chapter8. Shadow variable
## 8.1. Introduction
|中文名|英文名|注解|备注|
|-|-|-|-|
|真变量|genuine planning variable|```@PlanningEntity```|如VPR中的前停靠点previous standstill|
|影变量|shadow variable|```@***ShadowVariable```||
|真规划实体|genuine planning entity|```@PlanningEntity```|具有真变量和影变量。|
|影规划实体|shadow planning entity|```@PlanningEntity```|具有影变量，没有真变量|
以上中文名是我瞎翻译的。

影变量(shadow variable)的值可以通过真变量(genuine planning variable)计算出来，例如在VRP问题中，根据链路中的顾客实体的真规划变量——前停靠点(previous standstill)和距离矩阵，可以计算出车辆到每个停靠点的到达时间。  

影变量随真变量的改变而变化(求解器提供了VariableListener接口实时修改影变量)。含有影变量的类一定是规划实体类(记得在```<entityClass/>```中注册，并添加```@PlanningEntity```)。


## 8.2.Bi-directional variable(inverse relation shadow variable)
双向变量，逆关系影变量。
真规划实体的真规划变量是一个影规划实体，影规划实体的影规划变量(用@InverseRelationShadowVariable注解标记)是真规划实体。例如，cloud balancing问题中，Process类下的真规划变量是Computer类，同时，Computer类作为影规划实体，含有一个影规划变量processList列表，用来存储与Computer相关的Process。虽然，在optaplanner的Cloud balancing example中，作者并没有这样实现。同样在VRP问题中，Customer的前一个停靠点previousStandstill的逆关系影变量nextCustomer是Customer。作者在例子中也没有这样实现。

@InverseRelationShadowVariable(sourceVariableName = "?")
?处填真规划变量注解标记的get方法下的变量名。

第一个例子中真实体和影实体为多对一的关系，影实体中的容器不能为null，需要初始化为空容器。

一对一的问题要注意初始解要确定关系，能不能两者都为空就没有提了。

8.3.Anchor shadow variable
锚影变量，锚应该可以看做链表的初始点，在VRP问题中就是车。
@AnchorShadowVariable(sourceVariableName = "?")
这个注解写在真规划实体中返回锚的方法上。
?处填真规划实体的真规划变量注解标记的方法中返回的变量名。
在VRP问题中，真规划变量的变量名为previousStandstill，就填这个。
VRP问题中，各个对象的关系如下：
某个vehicle1的nextCustomer为customer1
customer1的真规划变量为vehicle，锚影变量为vehicle，nextCustomer为customer2
customer2的真规划变量为customer1，锚影变量为vehicle，nextCustomer为customer3
customer3的真规划变量为customer2，锚影变量为vehicle，nextCustomer为customer4
.
.
.
.
.
8.4.Custom VariableListener
在了解VariableListener的功能的过程中，发现自建项目时可以不用加以下注解。
@XStreamAlias("VrpVehicle")，在optaplanner的examples中，这个注解是为了读取optaplanner自带的XML格式的算例。

实现VariableListener接口来更新影变量。
第三个影变量注解：
@CustomShadowVariable
(
variableListenerClass = ?1.class,
Sources = {@PlanningVariableReference(variableName = “?2")}
)

以下为个人推测内容和疑问。从名称，参数和使用两个方面介绍。
(1)该注解名称为CustomerShadowVariable，是不是只能用在Customer类中或者是不是只能用在真规划实体中，如果我的项目中使用其他名称命名了真规划实体，那么@CustomerShadowVariable能不能用，可以用的话以Customer冠名又显得不伦不类。此外，可不可以用在影规划实体中。哪里都能用，就是这么不伦不类。
(2)该注解第一个参数用了本节的题目VariableListener。?1处写VariableListener接口的实现类，在该实现类中没有规划实体种类的要求，接口第二个参数仅仅要求传入一个Entity，那就有的想象了，传入影规划实体，是不是可以将该注解用在影规划实体的方法上。应该是注解所在的entity。不注册不更新。
(3)该注解第二个参数Sources，并用{}包围，找到了例子，需要写多个@PlanningVariableReference()，逗号隔开，()中可以填variableName = 字符串和entityClass = 类名.Class，也用逗号隔开。Reference没有找到用处，可能仅仅作为一个标记。仅有第二个参数时可以写：
@CustomShadowVariable(variableListenerRef = @PlanningVariableReference())。
Sources为触发器，其中变量变了就会改变。触发器所在的类不在该类下，需要注明。
(4)在optaplanner使用手册的例子用@CustomerPlanningVariable实现了@AnchorShadowVariable的功能。推测其也能实现@InverseRelationShadowVariable的功能。是一个比较通用的注解。
(5)有时候用在get方法上，有时候用在变量名上。
(6)如果俩影变量都变，则写在一个VariableListener中，效率比写俩高，仅第一个变量的注解中写variableListenerClass，另一个仅写reference，参考上一个shadow variable

引用类型需要加@DeepPlanningClone
Why is this needed? This is already a shadow var
如果仅仅为一个引用类型 vehicle = 索引1，复制后修改为索引2，对旧解无影响
如果是一个容器，C<> = 索引1，修改容器中的键值对，对旧解有影响，需要加深拷贝注解

Listener中before的用法，可以获取旧的信息并存储下来

8.5.VariableListener triggering order

The genuine and shadow variables form a graph, that determines the order in which the afterEntityAdded(), afterVariableChanged() and afterEntityRemoved() methods are called:
真变量和影变量构成图，变量之间的相关性，决定after*()方法的调用顺序。
同一个listener中的after方法的顺序应该是固定的，应该是listener顺序不同。
触发关系构成图，相互的关系可以调整。
顺序1：在所有的真变量变动后才会触发第一个listener。
顺序2：the last first shadow variable不懂这个是啥意思，感觉像是仅仅与已固定的变量相关的变量。这种变量变动后，触发第二个listener。以此类推

调用的相同VariableListener的顺序不能确定，如VRP中将线路1的某个点移动到线路2的某个点，将线路3的某个点移动到线路4的某个点，那么这两次移动的listener的顺序无法确定。

如果影变量在一次move中改变了两次，会调用两次VariableListener，两次都是监听初始的影变量，造成不必要的计算，requiresUniqueEntityEvents()重写该方法可以防止Listener多次调用，当求解器性能有略有变差。

在VRP问题中，Listener只使用了afterEntityAdd()和afterVariableChange()，它们都调用了同一个方法更新车辆路线信息，该方法中以传入的customer的previousStandstill为起始点更新信息，猜测：
afterEntityAdd()代表将customer加入新路径但未改变planningVariable的时点，因此获得的planningVariable为旧路径。
afterVariableChange()代表将customer加入新路径并改变planningVariable的时点，因此获得的planningVariable为新路径。
至此这两个方法完全覆盖了求解器对解的修改内容。
那所有的before*方法和afterRemove方法有啥用啊？

另外计算增量分数也用到after和before，带没有继承相同的接口

为什么有的加在变量上 有的加在方法上

Listener中修改影变量注意事项
scoreDirector.beforeVariableChanged(Object, String)
修改操作
ScoreDirector.afterVariableChanged(Object, String)
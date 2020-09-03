# 刚体

仅仅有物理世界，我们还是不能让游戏世界中的3D物体具有物理属性，为了让这些物体符合物理规则，我们还需要为其添加**刚体**。  

刚体是物理学中的一个概念，它可以看做是现实世界中真实物体的一个理想抽象，具备一些非常理想的物理性质，其中最重要的就是“运动中和受力后，形状和大小不变”，这大幅简化了物理模拟的计算模型。  

在Sein中，“刚体”通过组件[RigidBodyComponent](../../document/classes/rigidbodycomponent)体现，其提供了一系列参数来初始化和配置刚体信息。在**不考虑优化**的情况下，一般只需要通过以下代码为一个Actor添加刚体：  

```ts
actor.addComponent('rigidBody', Sein.RigidBodyComponent, {
  mass: 1,
  friction: .1,
  restitution: .1
});
```

参数中，`mass`是重量，`friction`是表面摩擦力，`restitution`是弹性系数，这些均为一个刚体的基本物理量。通过这几个参数为Actor添加了刚体组件，其就具备了产生物理效应的能力，从代码逻辑来看，也就是在物理世界中生成了与其一一对应的一个物体。在添加了组件之后，你也可以设置组件的一些属性例如`mass`、`friction`等来修改其物理属性，也可以通过调用一些方法比如`setLinearVelocity`、`getLinearVelocity`等来设置/获取像是线速度、角速度这样的状态。  

刚体组件在一个Actor中只能存在一个，在添加了刚体组件后，Actor可以直接通过`actor.rigidBody`来快速访问它。  

>注意，如果想使得两个刚体之间产生物理效应，**其中至少有一个刚体的质量不为0**。

## 优化

虽然通过上面三个简单的参数足以描述一个刚体，但如果所有的刚体都不加任何限制得添加，每一帧物理系统的计算量将很快膨胀到难以接受。所以Sein提供了一些优化的方法。  

优化的思想很简单。在游戏世界中的一段时间内，一般存在两种物体——会随着时间变动的和完全静止的。而对于会随着时间变动的物体，又分为完全受物理系统控制、完全受AI或玩家控制、以及两者的影响都受的。简单规约下来，其实我们只需要提供两个开关——一个决定游戏世界内的物体是否受物理世界内的刚体影响，一个反之。  

那么为什么是这两个开关呢？这就涉及到物理引擎的机制了。物理引擎的运行和游戏世界一样，也是在每一帧去计算刚体的物理效应以及之后的位置的。但物理世界和游戏世界毕竟是两个世界，虽然它们中的物体基本都存在一一绑定，但这绑定的仅仅是逻辑关系，我们还需要额外的同步逻辑来同步二者的`transform`。在Sein中，物理引擎在一帧的计算分为两步进行，其顺序如下：  

>从Actor向刚体同步Transform(1) -> 物理引擎计算刚体新的Transfrom -> 从刚体向Actor同步Transform(2)  

那么不难想到，如果一个物体是完全受物理引擎控制的，1中的计算就可以免除，若一个物体完全由AI或玩家控制，那么2中的计算就可以免除，若一个物体时完全静止的，那么二者的计算都可以免除。  

Sein中提供了`unControl`属性用于免除1中的计算，提供了`physicStatic`用于免除2中的计算，开发者可以直接在初始化参数中对其进行设置或者在后续通过对应属性进行设置：  

```ts
actor.rigidBody.unControl = true;
actor.rigidBody.physicStatic = true;
```

一般而言，对于完全静止的物体（比如地面、墙壁等），建议将这两个参数都设为`true`，这样能够大幅提升性能。

## Sleep/Wake/ForceSync

除了以上两个参数，刚体组件还提供了一些方法做更细致的控制。`sleep`和`wake`这一对方法用于使一个刚体进入“睡眠”状态或者“唤醒”它。如果一个刚体进入睡眠状态，那么在物理引擎计算时将会完全忽略它（“拾取”除外，这个下一节会说到），这个可以用于一些特殊目的，**比如不需要物理模拟，仅仅用于拾取的场合**。  

`forceSync`方法则提供了一个方式来强制同步刚体和Actor，这个也在某些场合有用。

## 分组和过滤

为了更加精细的控制，刚体组件也提供了类似图层一样的分组策略。你可以使用一个32bits的整形`filterGroup`来对一个刚体进行分组，进行分组后，可以给另一个刚体组件设置32bits的整形参数`filterMask`来决定是否要和这个刚体产生物理效应，这个判断结果是由位与运算`&`决定的。  

举个例子，比如我有A、B、C三个Actor，它们分别有刚体组件`rA`、`rB`、`rC`，其中我们设置了`ra.filterGroup = 0b01`、`rb.filterGroup = 0b10`，之后设置`rc.filterMask = 0b01`。那么在实际的物理计算中，`rc`会忽略`rb`，只会和`ra`进行计算，倘若想让`rc`和二者都进行计算，只需要保证其`filterMask`最低两位为`0b11`即可。

## 停止/恢复运作

有时候我们需要让物理刚体暂时停止运作，比如在其挂载的Actor被隐藏或者`UnLink`的时候。Sein提供了`rigidBody.disable`和`rigidBody.enable`方法来达成目的。  

实例可见[物理系统-停止运作](../../example/physic/disable)，注意在父级Actor`UnLink`的时候会默认触发`disable`，在`ReLink`的时候则会默认触发`enable`。但在改变`visible`的时候不会有任何变化，需要的话要自己触发。
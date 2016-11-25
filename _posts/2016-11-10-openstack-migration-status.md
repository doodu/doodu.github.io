# openstack instance migraiton status



openstack 虚拟机迁移总共有4种类型：

* resize (调整虚拟机配置大小)
* migraiton (冷迁移, 与resize动作基本一样)
* live-migration (热迁移)
* evacuation (疏散，用于将出问题的物理机上的虚机迁移到新的物理机上)

其中migration底层调用的就是resize的代码，在新旧的物理机和都会保留虚机的数据，用于等待确认或取消操作。

openstack 涉及到三种状态，vm_state，task_state和power_state。我们主要将一下在做迁移操作时涉及的task_state.

## 1. resize/migration 
在虚拟机active和stopped状态下可以进行migrate/resize，migration和resize的代码一样,resize与migrate的区别是在迁移的同时，改变虚拟机的flavor，所以他们的task_state都是一样的。他们的状态分为过渡状态和稳定状态。

迁移之后，会在数据库中新增migration表记录，同时会增加instance_system_metadata表记录，记录老的flavor和新的flavor，如果是resize，也会更新instances表中虚拟机的flavor信息。

对于resize，还有两个相关操作：confirm_resize和revert_resize，两个操作都是在虚拟机状态为resized下的操作。前者是resize的确认，后者是回退。也就是说，在做完migrate/resize操作后，给用户提供了后悔的机会，比如用户可能对新规格的虚拟机性能不满意，不愿意花冤枉钱，于是可以调用revert_resize，是虚拟机回退到之前的状态（包括虚拟机所在的主机和虚拟机的规格）。还需要注意，会有这样的情况，用户做完了resize/migrate，然后没有确认也没有回退（因为可能resize操作并不收费，所以用户想着能体验多长时间就用多长时间）。此时，OpenStack提供了循环任务的机制处理这种情况，当系统中配置项resize_confirm_window大于0时，计算节点会自动将在本机中状态为resized，时间超过resize_confirm_window的虚拟机进行确认。

**注意：resize/migration操作错误之后不会在migration数据表中出现error状态，任务卡死了（目前发现是这样的），所以当动作的过度状态长时间在数据库中，说明操作出错！！！**

状态转换图如下：

```flow
st=>start: 开始
e=>end: 结束
op1=>operation: pre-migrating
op2=>operation: migrating
op3=>operation: post-migrating
op4=>operation: finished
op5=>operation: confirming
op6=>operation: confirmed
op7=>operation: reverting
op8=>operation: reverted
c1=>condition: confirm?

st->op1->op2->op3->op4->c1
c1(yes,right)->op5->op6->e
c1(no)->op7->op8->e

```
pre-migrating --> migrating --> post-migrating --> fininshed --> confirm or revert


配置项allow_resize_to_same_host表示是否允许迁移到本机，默认是False。
在做migrate/resize前还应注意，计算节点间是需要配置nova用户无密码访问的；同时，migrate/resize操作支持共享存储。


## 2. live-migration
migrate操作会先将虚拟机停掉，也就是所谓的“冷迁移”，而os-migrateLive是“热迁移”，虚拟机不会停机。
live-migration是在虚拟机active状态下的操作，且不允许热迁移到本机。

live-migration允许用户指定block_migration参数，表示是否进行块迁移。有几种情况：

1. block_migration=True，但系统使用了共享存储，抛异常；
2. block_migration=False，但系统没有使用共享存储且虚拟机没有使用后端系统卷，抛异常；

live-migration 的task_state如下：

```
st=>start: 开始
e=>end: 结束
op1=>operation: running
op2=>operation: completed
op3=>operation: error/failed
c1=>condition: 正确?

st->op1->c1
c1(yes, right)->op2->e
c1(no)->op3->e
```


参考资料：

- http://www.cnblogs.com/starof/p/4221270.html
- http://blog.csdn.net/lynn_kong/article/details/9186201



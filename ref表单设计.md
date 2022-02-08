# ref表单设计

## 一、现状

### 1、渲染问题

> 对于antd3.0来说，每次操作form的单项Item就会重新渲染整个form

![img](https://raw.githubusercontent.com/caifeng123/pictures/master/v2-e1b71b79bb474b62378bdcf7f692336b_b.webp)

但实际上我们不应该每次修改都去让整个form变化，否则当渲染组件一多，消耗不必说，甚至会出现卡顿体验极差。

### 2、代码量问题

> 在业务中表单项往往很多，因此需要多个FormItem。代码就会变得很长很长，配置式就变得十分必要，可扩展性&代码整洁度&可读性都是很关键的

​	*老写法*

![image-20220128094606469](https://raw.githubusercontent.com/caifeng123/pictures/master/image-20220128094606469.png)

​	*新写法*

![image-20220128100907185](https://raw.githubusercontent.com/caifeng123/pictures/master/image-20220128100907185.png)

​	*样子*

![image-20220128100712681](https://raw.githubusercontent.com/caifeng123/pictures/master/image-20220128100712681.png)



## 二、方案

### 1、渲染

- 问题

  - 显而易见需要将每一项FormItem进行分离

  - 且每次独立操作都不应该直接去变化父组件属性，应当只修改子组件属性

  - 将组件粒度细化，并兼容嵌套表单、表格嵌套表单等操作
  - 连锁反应，操作A表单项，E、F表单项触发其他改变

- 解决方案

  - 全局使用ref作为数据管理工具

  - 和原先一样针对每个formItem外层包裹一层HOC，将方法进行注入。

    唯一不同的是在hoc中去使用`const [, setstate] = useState({});`实现强制更新

    因此更新一直是影响最小粒度的formItem

  - 自定义组件情况很多，会沿用之前的提供value、onChange等属性，让用户更容易实现自定义【嵌套表单、表格嵌套】下面会有demo

  - 对于之前版本使用全局更新，整体重新渲染，连锁反应在渲染时做判断很容易实现

    现在使用ref后，当前改变只会影响当前。A -> E\F\G组件，多播的概念就立刻跳了出来。

- 多播

  > 显而易见就是一对多/多对多的关系

![Multicast.svg](https://raw.githubusercontent.com/caifeng123/pictures/master/1920px-Multicast.svg-20220128112908098.png)

- 解决方案

  - 1、使用 proxy 

    - 每一项都可能是触发多播对象，因此需要将每一项的value都变成proxy对象
    - 并在proxy对象中的handle - set中去修改会受影响的组件(value)

    **问题**

    - 用户在编写自定义组件时，可能会多次注册影响组件
    - 最关键是会出现proxy函数设置先后问题的出现 

    > e.g. 当前有ABC三个组件。A -> B ->C->A 可能会出现循环影响的情况，proxy需要多次注册，每调用一次就需要重新注册

  - 2、使用响应式编程 - rxjs
    - 响应式编程，天生是为了这种场景而生
    - 在hoc处每个formItem项添加上发送事件，只有订阅处才会收到
    - 只需要在受影响组件注册订阅事件，订阅事件能拿到form内的所有值，无需两侧都挂载订阅需求



### 2、代码量

> 对于formItem来说，实际上代码重复率很高，通过上面的例子就能看出，复用复用！

相信大家，写表单代码都很多，自己都有自己的一套简化方法。写成配置式，基于类型循环表单项即可。往往对于简单的独立表单项来说这是可以的，可对于复杂联动表单项，会出现相互影响的情况的时候，每次都会做不必要的重复渲染。

因此若是基于原先antd3.0来看的话，都无法去避免一个情况，那就是重复渲染。碰到复杂联动表单项可能就会出现下面这种情况。

![未命名文件 (6)](https://raw.githubusercontent.com/caifeng123/pictures/master/%E6%9C%AA%E5%91%BD%E5%90%8D%E6%96%87%E4%BB%B6%20(6).png)

- 『连锁反应实现』A -> B，A的值会导致B的渲染不同，一般是写在B的effect中，当A的值变化再次调用onChange,导致渲染两次
  - 用户变化 触发组件A的onChange事件 导致form对象重新渲染
  - 当渲染完B组件时会触发useEffect 监听A的值再次调用onChange改变
  - 因此若使用配置项去循环渲染将会出现以下情况，若连锁反应越多 A->B->C->A 渲染次数将越来越多，这也是我们很多同学写的form表单在顶层console时发现渲染次数特别特别多

![未命名文件 (5)](https://raw.githubusercontent.com/caifeng123/pictures/master/%E6%9C%AA%E5%91%BD%E5%90%8D%E6%96%87%E4%BB%B6%20(5).png)



### 3、渲染演示

![1](https://raw.githubusercontent.com/caifeng123/pictures/master/1.gif)



## 三、设计

> 写前设计是最最重要的，设计好、返工少、写的快、bug少

### 1、功能

- 可配置式实现联动表单项
- 能按每行数量、冒号与表单项间隔、是否有表单描述项、是否有冒号来渲染
- 实时校验。初始时不校验，点击提交、修改单项时校验。自定义校验规则
- 自定义组件报错信息样式、组件值渲染
- 点击提交，会全部校验，将不正确的校验显示，并阻止提交

### 2、设计图

**\<RefItem />**

> 作为表单的入口，控制表单渲染样式和内容

- `options` - 通过配置表单项渲染列表

  其他字段用来渲染表单项排列方式与样式 以下字段可便捷调整

  | Property            | Description                                                 | Type                         | Default  |
  | ------------------- | ----------------------------------------------------------- | ---------------------------- | -------- |
  | labelColSpan        | 字段与组件间距                                              | number                       | 3        |
  | withColon           | label后是否要跟冒号                                         | Boolean                      | true     |
  | colCount            | 一行几列                                                    | number                       | 2        |
  | position            | 单个表单项在固定空间中所处位置                              | 'start' \| 'center' \| 'end' | 'start'  |
  | contentDisplay      | 内容表现形式<br> inline - 元素本身大小 block - 撑满剩余大小 | 'inline' \| 'block'          | 'inline' |
  | customItemClassName | 自定义样式                                                  | String                       | ''       |

> 其他属性将会透传到自定义组件中

- 对 `form` 注册全局校验函数 `validateFields` 在提交时调用

![validator](https://raw.githubusercontent.com/caifeng123/pictures/master/validator.png)



**\<ComponentWrapper />**

> 作为用户的自定义组件的HOC，为自定义组件添加了丰富的表单功能

| Property     | Description          | Type | Default |
| ------------ | -------------------- | ---- | ------- |
| onChange     | 表单的变化函数       |      |         |
| value        | 表单项的值           |      |         |
| error        | 表单项的错误         |      |         |
| validate     | 自定义校验函数       |      |         |
| getError     | 获取其他表单项错误值 |      |         |
| getValue     | 获取其它表单项的值   |      |         |
| keyName      | 表单keyName          |      |         |
| subject      | 用作监听特定项       |      |         |
| setFormValue |                      |      |         |



- 校验:
  - 默认按照 options 配置的 rules 校验
  - 可在自定义组件中调用 `validate` 自定义校验函数
  - 用户可以设定启用/关闭校验

- 修改值
  - **只会** 重新渲染 **当前组件** & **订阅当前组件值的组件**

- 添加自定义组件功能函数
  - 自由获取表单某项值函数
  - 自由获取表单某项错误函数
  - 自由修改表单某项值函数【不应该使用】

 5、默认注入onChange 和 value 适用于所有规范组件


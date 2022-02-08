## react 业务向开发串讲

### 基础钩子

所有demo： https://codesandbox.io/s/github/caifeng123/cchooks

1、useEffect 使用教学

[demo网址/TEACHDEMOS/UseEffectDemo](https://codesandbox.io/s/github/caifeng123/cchooks?file=/src/teachDemos/useEffect/index.tsx)

> 依赖项浅比较 使用 `Object.is()` 浅比较
>
> ```js
> // Object.is()
> function is(x,y){
>     return ( 
>       	(x===y&&(x!==0||1/x===1/y)) || (x!==x&&y!==y)
>    )
> }
> ```

```tsx
import { useEffect, useState } from "react";
import { Button } from "antd";

const SonDemo = ({ fatherA }: any) => {
  const [a, setA] = useState({ 2: 3 });
  const [b, setB] = useState({ 1: 2 });
  const c = [a, fatherA];

  useEffect(() => {
    console.log("init");
    return () => console.log("finish");
  }, []);

  useEffect(() => {
    console.log("a change");
  }, [a]);

  useEffect(() => {
    console.log("b change");
  }, [b]);

  useEffect(() => {
    console.log("everytime");
  }, [c]);

  useEffect(() => {
    console.log("father / a change");
  }, [...c]);

  return (
    <>
      打开console，随意感受下怎么会造成渲染~
      <br />
      {/* a:{a} */}
      <Button onClick={() => setA((a) => ({ ...a }))}>+a </Button>
      <br />
      b:{JSON.stringify(b)}
      <Button onClick={() => setB({ 1: b[1] + 1 })}>b["1"] + 1 </Button>
    </>
  );
};

export default () => {
  const [fatherA, setFatherA] = useState(0);
  return (
    <>
      <Button onClick={() => setFatherA((x) => x + 1)}>改fatherA</Button>
      <SonDemo fatherA={fatherA} />
    </>
  );
};
```



2、memo、useCallback 使用demo

[demo网址/TEACHDEMOS/MemoDemo](https://codesandbox.io/s/github/caifeng123/cchooks?file=/src/teachDemos/memo/index.tsx)

```tsx
import { useState, memo, useCallback } from "react";

const ExpensiveTree1 = ({ onClick }: { onClick: (e: number) => void }) => {
  const [a, A] = useState(1);
  const dateBegin = Date.now();
  // 很重的组件，不优化会死的那种，真的会死人
  while (Date.now() - dateBegin < 600) {}

  console.log("Render ExpensiveTree1 --- DONE");
  const handleClick = () => {
    A((a) => a + 1);
    onClick(a);
  };
  return (
    <div onClick={handleClick}>
      <p>很重的组件，不优化会死的那种(点我看控制台打印)</p>
    </div>
  );
};

const ExpensiveTree2 = memo(({ onClick }: { onClick: () => void }) => {
  const [a, A] = useState(1);
  const dateBegin = Date.now();
  // 很重的组件，不优化会死的那种，真的会死人
  while (Date.now() - dateBegin < 600) {}

  console.log("Render ExpensiveTree2 --- DONE");
  const handleClick = () => {
    A((a) => a + 1);
    onClick(a);
  };
  return (
    <div onClick={handleClick}>
      <p>很重的组件，经过memo优化</p>
    </div>
  );
});

const Render1 = () => {
  const [text1, updateText1] = useState("Initial value");
  const onClick1 = (a: number) => {
    console.log("回调函数被点击" + a);
  };
  return (
    <>
      <input
        value={text1}
        onChange={(e) => {
          updateText1(e.target.value);
        }}
      />
      <ExpensiveTree1 onClick={onClick1} />
    </>
  );
};

const Render2 = () => {
  const [text2, updateText2] = useState("Initial value");
  const onClick2 = () => {
    console.log("回调函数被点击");
  };
  return (
    <>
      <input
        value={text2}
        onChange={(e) => {
          updateText2(e.target.value);
        }}
      />
      <ExpensiveTree2 onClick={onClick2} />
    </>
  );
};

export default () => {
  return (
    <>
      <Render1 />
      <Render2 />
    </>
  );
};
```



### 自定义钩子介绍

Demo尝试：[hooks-demo](https://codesandbox.io/s/github/caifeng123/cchooks)

| 名称              | 作用                          | 举例                                                         |
| ----------------- | ----------------------------- | ------------------------------------------------------------ |
| UseCascadeDemo    | 控制级联输入                  | 需要依次选择从哪个国家、哪个城市和哪个地区或者哪个机场选择出发地点。 |
| useConcurrent     | 处理并发请求                  | 在多个图表时常会要求分多个 api 进行调用请求，但相互没有关联  |
| useCounter        | 数数钩子                      | 当一个页面需要多个计数器时，会将状态搞得特别多繁杂，因此使用钩子将增减赋值变为函数形式调用，简洁+减少代码量 |
| useDeepEffect     | useEffect 增强版              | 依赖项为复杂类型，数据庞大时使用                             |
| useEffectCallback | 保证唯一指向 usecallback 钩子 | 为 memo 包裹的厚重组件提供函数的唯一指向                     |
| UseRequest        | 远程请求钩子                  | 简化请求操作                                                 |
| UseSetState       | 对 setstate 的增强            | 1、能直接赋值单独属性 形如 class 的 setState 2、能对深度对象进行改变数据 如 setA({"name.hah": 2}) 3、能像原始的 setState 使用回调函数进行赋值 |
| UseDeepValue      | 深比较值索引                  | 当对象/数组不想因为其他 state 改变导致的组件渲染反复重新生成新的对象指针 （memo 时需要减少变化）时使用 |



### 库介绍

> 基于 @baidu/fe-components-v2 开发的高阶组件库 [@baidu/fe-components-hoc](https://console.cloud.baidu-int.com/devops/icode/repos/baidu/personal-code/HOC/tree/master)
>
> 将hooks 和 高阶组件组合打包
>
> 使用webpack5 + ttsc + babel 打包



### 如何打包开发库

[打包ts库流程](http://agroup.baidu.com/iotviz/md/article/4634059)
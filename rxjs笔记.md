# rxjs笔记

官网：[rxjs7](https://rxjs.dev/)

学习：[RXJS5](https://rxjs-cn.github.io/RxJS-Ultimate-CN/content/observer.html)

使用入门：[30天学会rxjs](https://blog.jerry-hong.com/series/rxjs/thirty-days-RxJS-07/)

## Observable

### 概念

> 可以持续生成数据的生产者
>
> 可以把Observable想象成去饭店消费排队的客人，都是过来接受服务的并且大家来的时间不能确定

总的来说，只要搞懂，如何去创建、订阅、执行数据队列这几个玩意就入了门，那么我们来看看这些东西的真面目吧。



### 创建

> 创建的方式多种多样，有创建固定个数的(create、构造函数、of、from)方法，也有生成无限个数的方法（事件监听、interval）

```js
// 0、普通的create创建【已废弃】
let stream$ = rxjs.Observables.create((observer) => {
  observer.next(4);
  observer.next(1);
  observer.error('some error');
  observer.complete();
})

// 1、通过构造函数创建 【替代方法】
const observable = new Observable(subscriber => {
  subscriber.next(1);
  subscriber.next(2);
  subscriber.next(3);
  setTimeout(() => {
    subscriber.next(4);
    subscriber.complete();
  }, 1000);
});

// 2、点击流
const click = fromEvent(document, "click", {
  capture: true
});
const positions = click.pipe(map((ev) => ev.clientX));
positions.subscribe((x) => console.log(x));

// 3、创建函数 具体看官网
// of(1,2,3)
// from([1,2,3])
// interval(1000)
// EMPTY
// defer(() => {isOk?of(1,2,3):interval(1000)}) 在需要时再去生成对应的 observable

// 轮询处理请求 demo
const obser = new Observable(obs => {
  setInterval(() => {
    fetch('https://codesandbox.io/s/gracious-hill-xnlxs?file=/src/index.ts').then(res => res.text())
    .then(res => obs.next(res));
  },2000)
})
obser.subscribe((e) => console.log(e))
```

数据流是有了，接下来就是需要如何去处理数据了。快马加鞭来看看订阅方法



### 订阅

> 简单来说就是对每个生产者的处理方法清单
>
> subscribe函数，分别对应 `Observer` next、error、complete 调用的三个方法

```js
observable.subscribe(fnNext, fnError?, fnComplete?)
```

延续饭店的例子，可以想象是处理排队顾客的小二，下一个正常客人来时执行(next)，遇到砸场子的客户执行(报错Error)，结束接客时执行(complete)【被砸场子和结束接客都将无法接收下一个客人了】。

```js
// 其中 stream$ 指的是 observable 对象
const sub = stream$.subscribe(
  (value) => console.log('Value', value),
  (err) => console.log('err', err),
  () => console.log('complete')
);
// or
const sub = stream$.subscribe({
  next: value => console.log('A next: ' + value),
  error: error => console.log('A error: ' + error),
  complete: () => console.log('A complete!')
});

// 为了健壮性来说，对于有注册的地方就必须要有对应的注销
setTimeout(() => {
  sub.unsubscribe();
}, 3000)
```

- 有时对于数据流有多个处理清单时，往往想要放在一起进行注销

```js
const sub1 = stream$.subscribe(next: value => console.log('A next: ' + value));
const sub2 = stream$.subscribe(next: value => console.log('A next: ' + value));

// 添加成捆绑操作
sub1.add(sub2);

// 此时注销将会将子Subscription也注销掉
sub1.unsubscribe() // sub2.unsubscribe()也会被执行

// 撤销捆绑
sub1.remove(sub2)
```

至此，我们有了数据流(创建)，也有了处理清单(订阅)，接下来直接跑就对了



### 执行

> 实际上跑也不是后面控制的，在创建时就已经设计好了情况

由于创建时区分了有限数据和无限数据那么情况也肯定是不一样的

- 有限数据 & 通过构造函数

可以手动主观的调用 上面清单中的 next error complete 方法【多是用来观察情况】

```js
let stream$ = new Observable((observer) => {
  observer.next(4);	// 4
  observer.next(1);	// 1
  observer.error('some error') // error: some error
  observer.complete();// complete
  observer.next(1);// （不运行）
})

stream$.subscribe({
  next: value => console.log('A next: ' + value),
  error: error => console.log('A error: ' + error),
  complete: () => console.log('A complete!')
});
```

- 其他数据

则无法手动触发next error complete 方法，按顺序执行next方法，报错时error方法处理，结束时调用complete方法(可能不存在结束)



## Operator

> 操作符，将数据流按操作符变化
>
> 延续上面的例子: 将来的顾客们进行些许变化，比如只能有钱人来吃(filter),吃的人得先去洗手(map),前5名客人(take),最后一个客人(last)

ps：operator忙忙多，很多情况都无法确定对应的operator，可用此判断 [rxjs决策树](https://rxjs.dev/operator-decision-tree)



### pipe方法

对于Observable来说，还有个不得不提的方法 pipe

> 对于数据流变化可能会有很多，拿上面的例子来说，按函数式的逻辑得写成  `take(map(filter()))` 的形式
>
> 『不优雅，难看』等字眼立马浮现。因此pipe管道的出现就是为了将所有的Operator进行串联，并以流的形式进行操作传递

```js
observable.pipe(
	filter(x => x.money === 'rich'),
  map(x => x.wash = true),
  take(5)
)
```



## Subject

> 常用于广播形式，可添加多个订阅方法，一次next触发。类似于看直播
>
> 相比于其他是1对1形式比如
>
> 每次subscribe都是最新的，类似于看视频

- 既能作为像之前Observable构造函数一样，能够调用.next调用下一个,进行广播

```js
// 普通的subject
const subject = new Subject();
const observerA = {
  next: value => console.log('A next: ' + value),
  error: error => console.log('A error: ' + error),
  complete: () => console.log('A complete!')
}

const observerB = {
  next: value => console.log('B next: ' + value),
  error: error => console.log('B error: ' + error),
  complete: () => console.log('B complete!')
}

subject.subscribe(observerA);
subject.subscribe(observerB);

subject.next(1);
// "A next: 1"
// "B next: 1"
subject.next(2);
// "A next: 2"
// "B next: 2"
```

- 也能作为observer被其他observable订阅

```js
import { Subject, from } from 'rxjs';
 
const subject = new Subject<number>();
 
subject.subscribe({
  next: (v) => console.log(`observerA: ${v}`)
});
subject.subscribe({
  next: (v) => console.log(`observerB: ${v}`)
});
 
const observable = from([1, 2, 3]);
 
observable.subscribe(subject); // You can subscribe providing a Subject
 
// Logs:
// observerA: 1
// observerB: 1
// observerA: 2
// observerB: 2
// observerA: 3
// observerB: 3
```

注意：使用情况条件要判断好：『若每次请求结果都不同，则需要使用subject』

> 一般搭配connect/share [operaters] 去搭配操作源 ps: 之前广播使用的 multicast(operator) - 已被废弃【迭代太快了】

- 对于connect来说，接收多个observer

```js
// 函数的参数是一个observable，可以将广播数据进行多种方式变化
const result = interval(1000).pipe(
  take(6),
  map((x) => Math.random()), // side-effect
  connect(
    (item) => merge(
      item.pipe(map((x)=>"A: " + x)),
      item.pipe(map((x)=>"B: " + x))
    )
  )
);

result.subscribe((x) => console.log(x));
```

- share() 等价于 `pipe(connect(() => new Subject()), refCount())` 

  `refCount` 使多播的 Observable 在第一个订阅者到达时自动开始执行，并在最后一个订阅者离开时停止执行。

```js
const result = interval(1000).pipe(
  take(6),
  map((x) => Math.random()), // side-effect
  share()
);

const subA = result.subscribe((x) => console.log("A: " + x));
const subB = result.subscribe((x) => console.log("B: " + x));
setTimeout(() => {
  subA.unsubscribe()
}, 2000)
```



## Scheuler

调度器，控制同步 & 异步调用

```js
// 调节可控成同步/异步
const observable3 = new Observable((observer) => {
  observer.next(1);
  observer.next(2);
  observer.next(3);
  observer.complete();
}) // 不加pipe这句话时，为同步 0 1 2 3 4
.pipe(observeOn(asyncScheduler)); // 此时为异步 0 4 1 2 3

console.log("0");
observable3.subscribe(subscribeObj);
console.log("4");
```



## Operator demo

两个用到的demo列举，其他的直接看 [弹珠图](https://rxmarbles.com/) / [决策树](https://rxjs.dev/operator-decision-tree)更快~

### takeUntil

> 两个observer，直到一个被触发时停止
>
> ![takeUntil marble diagram](https://raw.githubusercontent.com/caifeng123/pictures/master/takeUntil.png)

```js
const observable2 = interval(1000);
const click = fromEvent(document, "click", {capture: true});
const example = observable2.pipe(takeUntil(click))   
   
example.subscribe({
    next: (value) => { console.log(value); },
    error: (err) => { console.log('Error: ' + err); },
    complete: () => { console.log('complete'); }
});
```



### concatAll

> 将多维observable变为一维 看弹珠图即可了解
>
> ![concatAll marble diagram](https://raw.githubusercontent.com/caifeng123/pictures/master/concatAll.svg)

我们来模拟一下拖动事件

- 流程: 鼠标按下 -》 鼠标拖动 -》 鼠标抬起

- 三个observable，鼠标按下开始监听拖动事件，直到抬起结束监听拖动

- 对于mouseDown在concatAll前: 

  [[mouseMove1-1,...,mouseMove1-n], [mouseMove2-1,...,mouseMove2-n], [mouseMove3-1,...,mouseMove3-n]...]

- concatAll后: 

  [mouseMove1-1,...,mouseMove1-n,mouseMove2-1,...,mouseMove2-n,mouseMove3-1,...,mouseMove3-n,...]

```js
const mouseUp = fromEvent(body, "mouseup");
const mouseMove = fromEvent(body, "mousemove");
const mouseDown = fromEvent(dragDOM, "mousedown").pipe(
  map(() => mouseMove.pipe(takeUntil(mouseUp))),
  concatAll(),
  map((event) => ({ x: event.clientX, y: event.clientY }))
);

mouseDown.subscribe((pos) => {
  dragDOM.style.left = pos.x + "px";
  dragDOM.style.top = pos.y + "px";
  console.log(pos);
});
```
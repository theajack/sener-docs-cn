<!--
 * @Author: chenzhongsheng
 * @Date: 2023-05-14 13:15:35
 * @Description: Coding something
-->
# Router 中间件

## 使用

router中间件是最为重要也是最为基础的一个中间件，它的功能是用来进行路由分发

以下是一个简单的使用

```js
import { Sener, Router } from 'sener';

const router = new Router({
    '/demo': ({ query }) => {
        // or: 'get:/demo': ({ query }) => { // get: prefix can be ignored
        query.message = 'from get';
        return { data: query };
        // Custom headers or statusCode
        // return { data: query, headers: {}, statusCode: 200  };
    },
    'post:/demo': async ({ body }) => {
        body.message = 'from post'
        return { data: body };
    },
});

new Sener({
  port: 9000,
  middlewares: [router],
});
```

也可以使用json声明，以方便按模块定义路由规则，如果使用ts可以搭配接口使用

```ts
import { IRouter } from 'sener';
const user: IRouter = {/* ... */};
const comment: IRouter = {/* ... */};

const router = new Router(user, comment);
```

## 路由映射

Router 中间件接受一个路由映射，路由映射的值可以为函数或者对象

### key

映射的key值为路由的url，url的格式如下：

```
[MetaInfo][Method:]<Url>
```

1. MetaInfo

路由元信息为可选部分，如果有的话会被解析成context中的meta属性，可供所有的中间件来进行处理

meta的格式为 [name1=value1&name2=value2]，其中value部分如果为空，则会被赋值为 true，如 `[a&b=2]` 会被解析为 {a: true, b: '2'}

以下是一个代码示例

```js
const router = new Router({
    '[a&b=2]/demo': ({ meta }) => {
        // meta: {a: true, b: '2'}
    },
});
```

注：当路由映射的值为对象时，key上不需要加入meta部分

2. Method 为路由方法，也是可选参数，默认值为get，需要使用`:`分割

```js
const router = new Router({
    '/aa': (ctx) => {},
    'get:/bb': (ctx) => {},
    'post:/cc': (ctx) => {},
    'delete:/dd': (ctx) => {},
});
```

3. Url就是路由的path，该参数为必选

### value

路由映射的值可以使函数或对象，当为对象时，key中不可以加入meta部分，使用对象定义路由的好处是meta可以传入复杂类型的数据

```ts
// 当为函数时
export type IRouterHandler = (
    context: ISenerContext,
) => IPromiseMayBe<IHookReturn>; // IHookReturn 可以参考中间件章节

// 当为对象时
export interface IRouterHandlerData {
    handler: IRouterHandler;
    meta?: IJson;
    alias?: string[]; // 路由别名，用于将不同的url指向同一个路由
}
```

`IRouterHandler` 的参数是 context 对象，返回值即为中间件的返回值，处理逻辑与中间件一致

以下简单的例子

```js
const router = new Router({
    '/aa': (ctx) => {},
    '/bb': {
        meta: {},
        alias: ['/bb1', '/bb2'], // bb1 和 bb2 也会指向当前路由
        handler: (ctx) => {},
    },
});
```

## 私有路由

私有路由为一类特殊的路由，该种路由的入参与返回值与一般路有类似，区别是该种路由更类似于工具方法，不会被外部请求所访问，主要专门用于内部使用 `route` helper来调用

```js
const router = new Router({
    '#userCheck': (ctx) => {},
});
```

## 模糊匹配

Router 中间件支持模糊匹配，即可以匹配一类路由，然后在 handler 中根据参数进行分发


```js
const router = new Router({
    '/api/:userId': ({params}) => {
        return {data: params.userId};
    },
    // 正则条件
    '/api2/:userId(\d+)': ({params}) => {
        // 后跟括号表示需要参数满足正则表达式，否则路由会报错
        return {data: params.userId};
    },
    // 带#前缀表示数字
    '/api3/:#userId': ({params}) => {},
    // 带!前缀表示布尔类型
    '/api4/:!isTrue': ({params}) => {},
});
```


## 复用路由前缀

Router 中间件支持复用路由前缀，即可以将一类路由的前缀提取出来

使用 createRoute 方法来创建一组具有相同前缀的路由，子路由使用与完整路由一致，支持method前缀、私有路由、meta、模糊匹配等，使用方式如下

```js
import { createRoute } from 'sener';
const router = new Router({
    ...createRoute('/api/user', {
        '/info': ()=>{
            // ...
        },
        'post:/update': ()=>{
            // ...
        },
        // ...
    })
});
```

## router helper

router 中间件会注入以下 helper

```ts
interface IRouterHelper {
    meta: IJson; // 获取路由元信息
    params: IJson; // 获取路由模糊匹配中的参数
    index: ()=>number;
    route<T extends IHookReturn = IHookReturn>(
        url: string, data?: Partial<ISenerContext>,
    ): IPromiseMayBe<T>; // 调用其他路由，返回路由结果，一般用于复用路由逻辑
    redirect: (url: string, query?: IJson, header?: IJson) => void, // 路由重定向 （302）
}
```

### meta

meta在前文中已经做过了介绍，helper中的meta用来获取当前路由的元信息

```js
const router = new Router({
    '[db]/test': ({meta}) => {
        return {data: meta};
    },
});
```

### params

params在前文中已经做过了介绍，helper中的params是用来获取当前路由的参数

```js
const router = new Router({
    '/test/:id': ({params}) => {
        return {data: params};
    },
});
```

### route

route方法用来调用其他路由拿到返回结果，该方法可以访问私有路由

```js
const router = new Router({
    '#test': () => {},
    'get:/test': () => {},

    '/route-test': ({route}) => {
        return route('/route-test')
    },
    '/route-test2': ({route}) => {
        const data = route('/route-test');
        // ... do something
        return {data};
    },
});
```

### redirect

redirect方法用进行路由重定向

```js
const router = new Router({
    '/test1': () => {return {data: 'test1'}},
    '/test2': ({redirect}) => {
        return redirect('/test1')
    },
});
```

### index

index 方法会生成一个当前请求过程中递增的id，一般可以用来生成一些表示index的场景

```js
const router = new Router({
    '/test': ({index}) => {
        const data1 = {index: index()};
        const data2 = {index: index()};
        return success({data1, data2})
    },
});
```

## 工具方法

1. error & success
   
路由中可以使用 error 和 success 方法封装路由的返回值，定义如下

```ts
function error<T = null>(msg?: string, code?: number, data?: T): IMiddleWareResponseReturn<IRouterReturn<T>>
function success<T = any>(data?: T, msg?: string, extra?: {}): IMiddleWareResponseReturn<IRouterReturn<T>>
```

以下是一个简单的使用例子

```js
import {Router, error, success} from 'sener'
const router = new Router({
    '/test': () => {
        const data = something();
        if(data === null) return error();
        return success({data});
    },
});
```

2. responseXX 方法

路由handler中可以使用 responseXX 方法标识请求已经被响应，且返回值将响应结果注入 context 中。

```js
import {Router, error, success} from 'sener'
const router = new Router({
    '/test': ({send404}) => {
        const isLogin = something();
        if(!isLogin) {
            return responseXX();
        }
        return success({data});
    },
});
```

后续中间件遇到已被响应的标识之后会跳过hook，如果要处理已经标识过的请求，可以将 acceptResponded 值设置为 true。

```js
class CustomMiddle extends MiddleWare {
    acceptResponded = true;
    enter(ctx){
    }
}
```

3. markReturned 方法

如果第三方中间件已经自行使用 response 对象进行了发送请求响应，那么后续 sener 再次发送或者设置header会打印一个错误。

为了防止这种情况，可以使用 markReturned 方法表示请求已提前返回响应，不需要由sener统一发送响应。同时后续中间件遇到已被发送响应的标识之后会跳过hook，如果要处理已经标识过的请求，可以将 acceptReturned 值设置为 true。

```js
class CustomMiddle1 extends MiddleWare {

    enter({response, markReturned}){
        response.write('xxx');
        response.end();
        markReturned();
    }
}
class CustomMiddle2 extends MiddleWare {
    acceptReturned = true;
    enter({response, markReturned}){
    }
}
```
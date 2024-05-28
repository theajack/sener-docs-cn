<!--
 * @Author: chenzhongsheng
 * @Date: 2023-05-14 14:49:08
 * @Description: Coding something
-->
# mysql中间件

## 安装使用

mysql中间件为独立中间件，需要单独安装使用

```
npm i sener-mysql
```

```js
import { Mysql } from 'sener-mysql';
new Mysql({
    // ...
});
```

## 基础使用

```js
import { Sener, Router } from 'sener';
import { Mysql } from 'sener-mysql';

const router = new Router({
    '/demo': async ({ querySql }) => {
        const {results, fields} = await querySql('select * from user')
        return { data: {success: true} };
    },
});

new Sener({
  middlewares: [router, new Mysql({
    host: 'localhost',
    user: 'me',
    password: 'secret',
    database: 'my_db'
  })],
});
```

## 构造参数

mysql中间件依赖第三方包 [mysql](https://www.npmjs.com/package/mysql), 具体构造参数可以参考 `mysql.createConnection` 的参数

## 自定义的context

```ts
interface IMysqlHelper {
  sql: <Model extends Record<string, any> = {
    [prop: string]: string|number|boolean,
  }>(name: string)=>SQL<Model>; // 用于拼接SQL语句
  _: typeof Cond; // 用于拼接sql语句的条件
  table: <T extends keyof (Tables) >(name: T)=> Instanceof<(Tables)[T]>;
  querySql: (sql: string|QueryOptions) => Promise<{
    results: any;
    fields: FieldInfo[];
  }>;
  mysqlConn: Connection;
}

const Cond: {
    eq(v: any): string; // =
    notEq(v: any): string; // <> (!=)
    gt(v: any): string; // >
    lt(v: any): string; // <
    gte(v: any): string; // >=
    lte(v: any): string; // <=
    bt(v1: any, v2: any): string; // between
    in(vs: any[]): string; // in
    like(v: string): string; // like
    null(): string; // is null
    notNull(): string; // is not null
}
```

QueryOptions、FieldInfo、 Connection 具体使用请参考 [mysql](https://www.npmjs.com/package/mysql)


### sql

sql 方法用于快捷拼接sql语句，支持链式调用，简单使用方式如下

```js
const router = new Router({
    '/demo': async ({ sql, _ }) => {
        const sqlStr = sql('user').select().where([
            { age: _.gt(18) }
        ]).sql;
        return { data: {sqlStr} };
    },
});
```

以下为sql方法的类型声明

```ts
interface ISQLPage {
    index?: number;
    size?: number;
}
type ICondition<Model> = ({
    [prop in keyof Model]?: any;
})[];
interface IWhere<Model> {
    where?: ICondition<Model>;
    reverse?: boolean;
}
declare class SQL<Model extends Record<string, any> = {
    [prop: string]: string | number | boolean;
}, Key = keyof Model> {
    private tableName;
    private sql;
    constructor(tableName: string);
    private reset;
    select(...args: Key[]): this;
    selectDistinct(...args: Key[]): this;
    private _select;
    orderBy<T = Key>(...args: T[]): this;
    orderByDesc<T = Key>(...args: T[]): this;
    groupBy(name: Key): this;
    insert(data: Partial<Model>): this;
    update(data: Partial<Model>): this;
    delete(): this;
    where(conditions?: ICondition<Model>, reverse?: boolean): this;
    deleteAll(): this;
    count(): this;
    sum(name: Key): this;
    avg(name: Key): this;
    min(name: Key): this;
    max(name: Key): this;
    get v(): string;
    page({ index, size, }?: ISQLPage): this;
}
```

where 方法

```js
where([
  {age: 18, height: 170},
  {age: 10, height: 130},
  {age: 12, height: [130, 140]},
])
// 以上语句表示 (age=18 and height=170) or (age=10 and height=130) or (age=12 and (height=130 or 140))

where([
  {age: 18, height: 170},
  {age: 10, height: 130},
  {age: 12, height: [130, 140]},
], true)
// 第二个参数表示反转and和or的逻辑（内部数组不反转）
// 则以上表示 (age=18 or height=170) and (age=10 or height=130) and (age=12 or (height=130 or 140))
```

### table

Table 为mysql数据表封装的一层数据抽象层，Table是一个类，有许多的封装好的操作表数据的方法，开发者可以直接使用，也可继承自Table封装自己的业务逻辑

1. 直接使用

```js
const router = new Router({
    '/demo': async ({ table }) => {
        const user = table('user');
        const result = await user.page({
          index: 3,
          size: 20,
        }).exec();
        return { data: {result} };
    },
});
```

以下为table的类型声明

```ts
declare class Table<Model extends Record<string, any> = Record<string, any>> {
    sql: SQL;
    helper: IMysqlHelper$1;
    allKeys: string[];
    constructor(name: string, target: Mysql$1);
    find(...conds: ICondition<Model>): Promise<Model | null>;
    exist(...conds: ICondition<Model>): Promise<boolean>;
    filter(...conds: ICondition<Model>): Promise<Model[]>;
    page(data?: ISQLPage & IWhere<Model> & {
        orderBy?: {
            keys: (keyof Model)[];
            desc?: boolean;
        };
    }): Promise<Model[]>;
    count(where?: ICondition<Model>, reverse?: boolean): Promise<number>;
    update(data: Partial<Model>, conds: ICondition<Model>, reverse?: boolean): Promise<any>;
    add(data: Partial<Model>): Promise<{
        affectedRows: number;
        insertId: number;
    }>;
    exec<T = any>(sql: SQL): Promise<IQuerySqlResult$1>;
}
```

2. 继承自定义业务逻辑

```ts
interface IUser {
    user_id: number,
    age: number,
    // ...
}
export class User extends Table<IUser> {
    async auth (ukey: string) {
      // todo
    }
    async login (data: any){
      // todo
    }
}
```

## ts类型声明

通过泛型传入数据类型之后，后续table方法的参数和返回值都会有相应的类型支持

示例如下

```ts
interface IUser {
    user_id: number,
    age: number,
    // ...
}
class User extends Table<IUser> {
    async auth (ukey: string) {
      // todo
    }
    async login (data: any){
      // todo
    }
}
const tables = {
    user: User,
    // ...
};

type ITables = typeof tables;

declare module 'sener-extend' {
    interface Table {
        tables: ITables;
    }
}
```
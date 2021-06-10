<h1>LINQxy <sub>/linkzē/ (LINQ Proxy)</sub></h1>


> Write LINQ and get query objects of different kinds for different processors.

## What is LINQxy for?
LINQxy is a standard for data querying in TypeScript. It's a proxy translating LINQ queries into different query objects for different libraries (depends on plugins). You can write LINQ and use it to query MongoDB, TypeORM, ZangoDB (IndexDB) or anything else.

## How the LINQxy Looks Like?
Lets have an interface of the data record we want to query.
```typescript
enum Gender {
  Female,
  Male,
  Unspecified,
}

interface IPerson {
  name: string;
  age: number;
  xp: number;
}

interface IEmployee extends IPerson {
  work: string;
  salary: number;
}
```
And a generic usage.
```typescript
import { Queryable } from "linqxy";

const someFilterObject = { workDescKeywords: [ "javascript", "typescript" ] };

const empQuery = Queryable.create<IEmployee>()
  .filter(emp => emp.xp > 5 && !emp.name.startsWith("Jo"))
  .filterIf(emp => someFilterObject.workDescKeywords.contains(emp.work), !!someFilterObject.workDescKeywords)
  .map(emp => ({
    name: emp.name,
    xp: emp.xp
  }));
```

Result of the `empQuery` variable is Queryable object which can be build into eg. TypeORM.
Same example with TypeORM plugin will look like:

```typescript
import { Queryable } from "linqxy-typeorm";
import Employee from "./Employee";

const someFilterObject = { workDescKeywords: [ "javascript", "typescript" ] };

const emps = await Queryable.create(Employee)
  .filter(emp => emp.xp >= 5 && !emp.name.startsWith("Jo"))
  .filterIf(emp => someFilterObject.workDescKeywords.contains(emp.work), !!someFilterObject.workDescKeywords)
  .map(emp => ({
    name: emp.name,
    xp: emp.xp
  }))
  .find();
```

Which is translated into query:
```typescript
await connection.getRepository(Employee).find({
    xp: MoreThanOrEqual(5),
    name: Not(Like("Jo%")),
    work: In(["javascript", "typescript"])
});
```

### How Does It Work?
Sorry for JavaScript users, but LINQxy uses TypeScript transformer [typescript-expression-transformer](https://github.com/Hookyns/expression-transformer). See the repo of the expression transformer for more information.

Is there a volunteer to create the Babel transform plugin?

## Synopsis
```typescript
class Queryable<TRecord> 
{
  filter(Expresson<(r: TRecord) => boolean> expression): Queryable<TRecord> {}
  
  filterIf(Expresson<(r: TRecord) => boolean> expression, condition: boolean): Queryable<TRecord> {}
  
  innerJoin<T, R>(
    target: T[] | Queryable<T>, 
    on: Expression<(first: TRecord, second: T) => boolean>, 
    result: Expression<(first: TRecord, second: T) => R>
  ): Queryable<TRecord> {}
  
  map<R>(Expresson<(r: TRecord) => R> expression): Queryable<R> {}
}
```

## Plugins
Plugins must implement IQueryGenerator interface. See wiky. **TODO**

Optionaly, it can extend Queryable object and provide some extra functionality.
Eg. like TypeORM plugin:

```typescript
import { Queryable as BaseQueryable} from "linqxy";
import { Generator as TypeORMGenerator } from "./Generator";

class Queryable<TRecord> extends BaseQueryable
{
  static create<TEntity>(type: typeof T): Queryable<TRecord> {}
  
  async find(): Promise<Array<TRecord>> {
    return await connection.getRepository(this.RecordType).find(this.build(TypeORMGenerator));
  }
  
  async findOne(): Promise<TRecord> {}
}

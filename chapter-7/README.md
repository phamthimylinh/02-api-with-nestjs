When we build an application, we create many entities. They often somehow relate to each other, and defining such relationships is an essential part of designing a database. In this article, we go through what is a relationship in the context of a Postgres database and how do we work with them using TypeORM and NestJS.

The relational databases have been around for quite some time and work great with structured data. They do so by organizing the data tables and linking them to each other. When running various SQL queries, we can join the tables and extract meaningful information. There are a few different types of relationships, and today we go through them with the use of examples.

> We've also gone through it in the [Typescript Express series](http://wanago.io/2019/01/21/typescript-express-tutorial-8-types-of-relationships-with-postgres-and-typeorm/). The below article acts as a recap of what we can get from there. This time we also look more into the SQL queries that TypeORM generates.

You can find all of the code from this series in [this repository](https://github.com/mwanago/nestjs-typescript).

# One-to-one
With the **one-to-one** relationship, the first table has one matching row in the second table, and vice versa.
The most straightforward example would be adding an **address** entity.

>> users/address.entity.ts
```typescript
import { Column, Entity, PrimaryGeneratedColumn } from "typeorm";

@Entity()
class Address {
    @PrimaryGeneratedColumn()
    public id: number;

    @Column()
    public street: string;

    @Column()
    public city: string;

    @Column()
    public country: string;
}
export default Address;
```

Let's assume that6 one address can be linked to just one user. Also, a user can't have more than one address.
To implement the above, we need a **one-to-one** relationship. When using TypeORM, we can create it effortlessly with the use of decorators.

>> users/user.entity.ts
```typescript
import { Column, Entity, PrimaryGeneratedColumn, OneToOne } from "typeorm";
import { Exclude } from "class-transformer";
import { Address } from "./address.entity";

@Entity()
class User {
    @PrimaryGeneratedColumn()
    public id: number;

    @Column({ unique: true })
    public email: string;

    @Column()
    public name: string;

    @Column()
    @Exclude()
    public password: string;

    @OneToOne(() => Address)
    @JoinColumn()
    public address: Address;
}
export default User;
```
Above, we use the `@OneToOne()` decorator. Its argument is a function that returns the class of the entity that we want to make a relationship with.
The secon decorator, the `JoinColumn()`, indicates that the `User` entity owns the relationship. It means that the reow of the `User` table contain the `addressId` column that can keep the id of an address. We use it only on one side of the relationship.

We can look into pgadmin to inspect what TypeORM does to create the desired relationship.

![one-to-one](https://wanago.io/wp-content/uploads/2020/06/Screenshot-from-2020-06-21-17-54-28.png)

Above, we can see that the `addressId` is a regular integer column. It has a **constraint** put onto it that indicates that any value we place into the `addressId` column needs to match some id in the `address` table.


The above can be simplified without the `CONSTRAINT` keyword.
```typescript
CREATE TABLE users(
    // ...
    addressId integer REFERENCES address(id)
)
```
Both `ON UPDATE NO ACTION` and `ON DELETE NOT ACTION` are a default behavior. They indicate that Postgres will raise an error if we attempt to delete or change the id of an address that is currently in use.

The `MATCH SIMPLE` refers to a situation when we use more than one column as the foreign key. It means that we allow some of them to be null.

## Inverse relationship






























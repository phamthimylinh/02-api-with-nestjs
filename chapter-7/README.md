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
Currently, our relationship is *unidirectional*. It means that only one side of the relationship has information about the other side. We could change that by creating an **inverse relationship**. By doing so, we make the relationship between the User and Address **bidirectional**.
To create the inverse relationship, we need to use the `@OneToOne()` and provide a property that holds the other side of the relationship.

>> users/address.entity.ts
```typescript
import { Column, Entity, OneToOne, PrimaryGeneratedColumn } from "typeorm";
import { User } from "./user.entity";

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

    @OneToOne(()=> User, (user: User)=> user.address)
    public user: User;
}
export default Address;
```
The crucial thing is that the **inverse relationship** is a bit of an abstract concept, and it does not created nay additional columns in the database.

![column for inverse relationship](http://wanago.io/wp-content/uploads/2020/06/Screenshot-from-2020-06-21-19-10-28-1.png)

Storing the information about both sides of the relationship can come in handy. We can easily relate to both sides, for example, to fetch the addresses with users.

```typescript
getAllAddressesWithUsers() {
    return this.addressRepository.find({relations:['user']})
}
```

if we want our related entities always to be included, we can make our relationship **eager**.
```typescript
@OneToOne(() => Address, {
    eager: true,
})
@JoinColumn()
public address: Address;
```
Now, every time we fetch users, we also get their addresses. Only one side of the relationship can be eager.

# Saving the related entities
Right now, we need to save users and addresses separately and this might not be the most convenient way. Instead, we can turn on the **cascade** option. Thanks to that, We can saving an address while saving a user.

```typescript
@OneToOne(() => Address, {
    cascade: true,
    eager: true,
})
@JoinColumn()
public address: Address;
```

![cascade option](https://wanago.io/wp-content/uploads/2020/06/Screenshot-from-2020-06-21-19-49-32.png)


# One-to-many and many-to-one
The **one-to-many** and **many-to-one** is a relationship where a row from the first table can be linked to multiple rows of the second table.
Rows from the second table can be linked to just one row of the first table.

The above is a very fitting relationship to implement to posts and users that we've defined in the previous parts of this series. Let's assume that a user can create multiple posts, but a post has just one author.

>> users/user.entity.ts
```typescript
import { Column, Entity, JoinColumn, OneToOne, OneToMany, PrimaryGeneratedColumn } from "typeorm";
import { Exclude } from "class-transformer";
import Address from "./address.entity";
import Post from "./post.entity";

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

    @OneToOne(() => Address, {
        eager: true,
        cascade: true,
    })
    @JoinColumn()
    public address: Address;

    @OneToMany(() => Post, (post: Post) => post.author)
    public posts: Post[];
}
export default User;
```
Thanks to using the `@OneToMany()` decorator, one user can be linked to multiple posts. We also need to define the other side of this relationship.

```typescript
import { Column, Entity, ManyToOne, PrimaryGeneratedColumn } from "typeorm";
import User from "./user.entity";

@Entity()
class Post {
    @PrimaryGeneratedColumn()
    public id: number;

    @Column()
    public title: string;

    @Column()
    public content: string;

    @Column({ nullable: true })
    public category?: string;

    @ManyToOne(() => User, (author: User) => author.posts)
    public author: User;
}
export default Post;
```

Thanks to `@ManyToOne()` decorator, many posts can be related to one user.

We implemented the authentication in the [third part of this series](http://wanago.io/2020/05/25/api-nestjs-authenticating-users-bcrypt-passport-jwt-cookies/). When a post is created in our API, we have access to the data about the authenticated user. We need to use it to determine the author of the post.

```typescript
@Post()
@UseGuards(JwtAuthenticationGuard)
async createPost(@Body() post: CreatePostDto, @Req() req: RequestWithUser) {
    return this.postsService.create(post, req.user);
}
```
```typescript
async createPost(post: CreatePostDto, user: User) {
    const newPost = await this.postsRepository.create({
        ...post,
        author: user
    });
    await this.postsRepository.save(newPost);
    return newPost;
}
```
If we want to return a list of posts with the authors, we can now easily do so.

```typescript
getAllPosts() {
    return this.postsRepository.find({ relations: ['author'] });
}
async getPostById(id: number) {
    const post = await this.postsRepository.findOne(id, { relations: ['author'] });
    if (post) {
        return post;
    }
    throw new PostNotFoundException(id);
}
```
![response one-to-many](https://wanago.io/wp-content/uploads/2020/06/Screenshot-from-2020-06-21-21-54-45.png)
 If we look into the database, we can see that the side of the relationship that  user `ManyToOne()` decorator stores the foreign key.
 ![table user](https://wanago.io/wp-content/uploads/2020/06/Screenshot-from-2020-06-21-21-54-45.png)
 This means that the post stores the id of the author and not the other way around.

# Many-to-many
Previously, we add a property called **category** to our posts. Let's elaborate on that more.
We would like to be able define categories reused across posts. We also want a single post to belong to multiple categories.
The above is **many-to-many** relationship. It happens when a row from the first table can be link to multiple rows from the second table and the other way around.

>> categories/category.entity.ts
```typescript
import { Column, Entity, PrimaryGeneratedColumn } from "typeorm";

@Entity()
class Category {
    @PrimaryGeneratedColumn()
    public id: number;

    @Column()
    public name: string;
}
export default Category;
```
>> posts/posts.entity.ts
```typescript
import { Column, Entity, JoinTable, ManyToOne, ManyToMany, PrimaryGeneratedColumn } from "typeorm";
import User from "./user.entity";
import Category from "./category.entity";

@Entity()
class Post {
    @PrimaryGeneratedColumn()
    public id: number;

    @Column()
    public title: string;

    @Column()
    public content: string;

    @Column({ nullable: true })
    public category?: string;

    @ManyToOne(() => User, (author: User) => author.posts)
    public author: User;

    @ManyToMany(() => Category)
    @JoinTable()
    public categories: Category[];
}
export default Post;
```
When we use `@ManyToMany()` and `@JoinTable()` decorators, TypeORM set ups and additional table. This way, neither the Post nor Category tables stores the data about the relationship.

```typescript
CREATE TABLE public.post_categories_category (
    "postId" integer NOT NULL,
    "categoryId" integer NOT NULL,
    CONSTRAINT "PK_91306c0021c4901c1825ef097ce" PRIMARY KEY ("postId", "categoryId"),
    CONSTRAINT "FK_1e4e2f1b1b2e3c2b4e3c2b4e3c2" FOREIGN KEY ("postId") 
        REFERENCES public.post (id) MATCH SIMPLE
        ON UPDATE NO ACTION
        ON DELETE NO ACTION,
    CONSTRAINT "FK_6a6b4b5d9b2e3c2b4e3c2b4e3c2" FOREIGN KEY ("categoryId") 
        REFERENCES public.category (id) MATCH SIMPLE
        ON UPDATE NO ACTION
        ON DELETE NO ACTION
)
```
Above we can see that our new `post_categories_category` table uses a primary key that consists of the `postId` and `categoryId`. It also uses the `postId` and `categoryId` combined.
![respone](https://wanago.io/wp-content/uploads/2020/06/Screenshot-from-2020-06-21-22-59-38.png)

we are also make the **many-to-many** relationship bidirectional. Remember to use the JoinTable decorator only on one side of the relationship, through.
```typescript
@ManyToMany(() => Category, (category: Category) => category.posts)
@JoinTable()

public categories: Category[];
```

```typescript
@ManyToMany(() => Post, (post: Post) => post.categories)
public posts: Post[];
```
Thanks to doing the above, we can easily fetch categories along with their posts.

```typescript

getAllCategories() {
  return this.categoriesRepository.find({ relations: ['posts'] });
}

async getCategoryById(id: number) {
  const category = await this.categoriesRepository.findOne(id, { relations: ['posts'] });
  if (category) {
    return category;
  }
  throw new CategoryNotFoundException(id);
}

async updateCategory(id: number, category: UpdateCategoryDto) {
  await this.categoriesRepository.update(id, category);
  const updatedCategory = await this.categoriesRepository.findOne(id, { relations: ['posts'] });
  if (updatedCategory) {
    return updatedCategory
  }
  throw new CategoryNotFoundException(id);
}
```
![response many-to-many relationship](https://wanago.io/wp-content/uploads/2020/06/Screenshot-from-2020-06-21-22-58-41.png)

# Summary
This time we've covered creating relationships while using NestJS with Postgres and TypeORM. It included one-to-one, one-to-many and many-to-many. We supplied them with various options, such as `cascade` and `earge`. We've also looked into SQL queries thay TypeORM create, understanding better how it works.

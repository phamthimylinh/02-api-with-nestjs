Authentication is a crucial part of almost every web application. There are many ways to approach it, and we've handled it manually in our [Typesript Express series](http://wanago.io/2018/12/24/typescript-express-registering-authenticating-jwt/). This time we look into the **passport**, which is the most popular Node.js authentication library. We also register users and make their passwords secure by **hashing**.

You can find all of the code from this series in [this repository](https://github.com/mwanago/nestjs-typescript). Feel free to give it a star.

# Defining the User entity
The first thing to do when considering authentication is to **register** our users. To do so, we need to define an entity for our users.

> users/user.entity.ts
```typescript
import {Column, Entity, PrimayGenaratedColumn } from 'typeorm';

@Entity()
class User {
  @PrimayGenaratedColumn()
  public id?: number;

  @Column({ unique: true})
  public email: string;

  @Column()
  public name: string;

  @Column()
  public password: string;
}

export default User;
```
The only new thing above is the **unique** flag. It indicates that there should not be two users with the same email.
This  functionally is built into PostgreSQL and helps us to keep consistency of our data. Later, we depend on emails being unique when authenticating.

We need to perform a few operations on our users. To do so, let's create a service.

> users/users.service.ts
```typescript
import { HttpException, HttpStatus, injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import User from './user.entity';
import CreateUserDto from './dto/createUser.dto';

@injectable()
export class UsersService {
  constructor(
    @InjectRepository(User)
    private usersRepository: Repository<User>
  ) {}

  async getByEmail(email: string) {
    const user = await this.usersRepository.findOne({ email });
    if (user) {
      return user;
    }
    throw new HttpException('User with this email does not exist', HttpStatus.NOT_FOUND);
  }

  async create(userData: CreateUserDto) {
    const newUser = await this.usersRepository.create(userData);
    await this.usersRepository.save(newUser);
    return newUser;
  }
}
```
> users/dto/createUser.dto.ts
```typescript
export class CreateUserDto {
  email: string;
  name: string;
  password: string;
}

export default CreateUserDto;
```
All of the above is wrapped using a module.

> users/users.module.ts

```typescript
import { Module } from '@nestjs/common';
import { UsersService } from './users.service';
import { TypeOrmModule } from '@nestjs/typeorm';
import User from './user.entity';

@Module({
  imports: [TypeOrmModule.forFeature([User])]
  providers: [UsersService],
  exports: [UsersService]
})

export class UsersModule {}
```

# Handling passwords






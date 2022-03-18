
Well I have successfully deployed a NestJS project on lambda, with RDS as database.

Here is how I achieved that

# Steps

1. [Initialized brand new NestJS Project](project-initialization)
1. [Configured typeorm]()
1. [Installed serverless]()
1. [Created new AWS user and gave policies]()
1. [Configured aws secrets on project]()
1. [Created serverless.yml with handler]()
1. [Created user entity with login and register]()
1. [Created RDS instance]()
    1. [Configured security group on RDS]()
1. [Called typeorm migration from local(bad practice but was pain to call from lambda)]()
1. [Deployed my project to lambda(called 1 command but it had a lot of steps)]()
1. [Assigned RDS secrets on lambda environment variables]()
1. [Trying the lambda function]()

# Project initialization

`nest new test-project`

# Typeorm configuration

<details>
<summary>ormconfig.js</summary>

```js
//ormconfig.js
module.exports = {
  type: 'mysql',
  host: process.env.DB_HOST,
  port: Number(process.env.DB_PORT),
  username: process.env.DB_USER,
  password: process.env.DB_PASS,
  database: process.env.DB_DATABASE,
  autoLoadEntities: true,
  synchronize: false,
  logging: false,
  entities: ['dist/**/*.entity.js'],
  migrations: ['dist/database/migrations/**/*.js'],
  subscribers: ['dist/subscribers/**/*.js'],
  cli: {
    migrationsDir: 'src/database/migrations',
  },
};
```
</details>

# Serverless installation

`npm i -g serverless`

# AWS Create user

1. Create user
1. Save access key and secret access key
1. Give that user a policy that is required to deploy
    1. Needs Cloudformation, S3, Lambda and lot other policies

# AWS configurations on project

There is 2 way to configure, I did the latter

1. create .aws folder in project like this example

```
// .aws/credentials
```

1. Configure through AWS CLI

# Create serverless.yml and handler

### serverless.yml
<details>
<summary>serverless.yml</summary>

```yml
//serverless.yml
service:
  name: service-name
frameworkVersion: '2'
plugins:
  - serverless-offline

provider:
  name: aws
  runtime: nodejs14.x
  stage: ${opt:stage, 'local'}
  region: ${opt:region, 'ap-southeast-1'}
  deploymentBucket:
    name: bucket-name
    serverSideEncryption: AES256

functions:
  main:
    handler: dist/serverless.handler
    environment:
      ENV: ${opt:stage, 'local'}
    events:
      - http:
          cors: true
          path: '/'
          method: any
      - http:
          method: any
          path: /{any+}
          cors: true
```
</details>



### handler
<details>
<summary> handler </summary>

```typescript
//src/main.ts
import { APIGatewayProxyEvent, APIGatewayProxyHandler } from 'aws-lambda';
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { Server } from 'http';
import { ExpressAdapter } from '@nestjs/platform-express';
import { createServer, proxy } from 'aws-serverless-express';
import * as express from 'express';
import { ValidationPipe } from '@nestjs/common';

let cachedServer: Server;

const bootstrapServer = async (): Promise<Server> => {
  const expressApp = express();
  const adapter = new ExpressAdapter(expressApp);
  const app = await NestFactory.create(AppModule, adapter);
  app.enableCors();
  app.useGlobalPipes(new ValidationPipe({ whitelist: true }));
  console.log('before app init');
  await app.init();
  return createServer(expressApp);
};

export const handler: APIGatewayProxyHandler = async (
  event: APIGatewayProxyEvent & { source?: string },
  context,
) => {
  console.log('in handler');
  if (event.source === 'serverless-plugin-warmup') {
    console.log('WarmUP - Lambda is warm!');
    return 'Lambda is warm!';
  }
  if (!cachedServer) {
    cachedServer = await bootstrapServer();
  }
  console.log('before promise');
  return proxy(cachedServer, event, context, 'PROMISE').promise;
};

```
</details>

# Login and register endpoint

created login and register endpoint for testing purpose

<details>
<summary>User entity</summary>

```ts
//user.entity.ts
import { Exclude } from 'class-transformer';
import { Entity, PrimaryGeneratedColumn, Column } from 'typeorm';

@Entity('users')
export class Users {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column({ nullable: true })
  name: string;

  @Column({ unique: true })
  email: string;

  @Column({ length: 60 })
  @Exclude()
  password: string;

  @Column({ nullable: true })
  phone_number: string;

  @Column({ default: false })
  @Exclude()
  isConfirmed: boolean;

  @Column()
  @Exclude()
  confirmToken: string;
}
```
</details>
<details>
<summary>Auth controller</summary>

```ts
//auth.controller.ts
import {
  Controller,
  Post,
  Body
} from '@nestjs/common';
import { plainToClass } from 'class-transformer';
import { Users } from 'src/entities/user.entity';
import { AuthService } from './auth.service';
import { LoginUserDto } from './dto/login.dto';
import { RegisterUserDto } from './dto/register-user.dto';

@Controller('auth')
export class AuthController {
  constructor(private readonly authService: AuthService) {}

  @Post('register')
  async register(@Body() registerUser: RegisterUserDto) {
    const user = await this.authService.register(registerUser);
    return plainToClass(Users, user);
  }

  @Post('login')
  async login(@Body() createAuthDto: LoginUserDto) {
    return await this.authService.login(createAuthDto);
  }
}
```

</details>

<details>
<summary>Auth service</summary>

```ts
//auth.service.ts
import {
  Injectable,
  NotFoundException,
  UnauthorizedException,
  UnprocessableEntityException,
} from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Users } from 'src/entities/user.entity';
import { Repository } from 'typeorm';
import { RegisterUserDto } from './dto/register-user.dto';
import { UpdateAuthDto } from './dto/update-auth.dto';
import * as bcrypt from 'bcryptjs';
import { JwtService } from '@nestjs/jwt';
import { LoginUserDto } from './dto/login.dto';
import { plainToClass } from 'class-transformer';

const ITERATIONS = 12;

@Injectable()
export class AuthService {
  constructor(
    @InjectRepository(Users)
    private readonly userRepository: Repository<Users>,
    private readonly jwtService: JwtService,
  ) {}
  async register(body: RegisterUserDto) {
    const user = await this.userRepository.findOne({
      where: { email: body.email },
    });
    if (user) {
      throw new UnprocessableEntityException('User already registered');
    }
    const password = await this.hashPassword(body.password);
    const confirmToken = this.jwtService.sign({ email: body.email });
    const res = await this.userRepository.save({
      ...body,
      password,
      confirmToken,
      name: body.email.split('@')[0],
    });
    return res;
  }

  async login(body: LoginUserDto) {
    const user = await this.findByEmail(body.email);
    const passwordIsValid = this.comparePassword(body.password, user.password);
    if (!passwordIsValid) {
      throw new UnauthorizedException('Wrong password');
    }
    const payload = {
      email: user.email,
      id: user.id,
    };
    const accessToken = this.jwtService.sign(payload);

    return { user: plainToClass(Users, user), metadata: { accessToken } };
  }

  public async findByEmail(email: string): Promise<Users> {
    const user = await this.userRepository.findOne({
      where: {
        email: email,
      },
    });

    if (!user) {
      throw new NotFoundException(`User not found`);
    }

    return user;
  }

  comparePassword(password: string, hash: string) {
    return bcrypt.compare(password, hash);
  }
  hashPassword(password: string) {
    return bcrypt.hash(password, ITERATIONS);
  }
}
```
</details>

# Creating RDS instance

## Configure security group

# Typeorm migration

Generate new migration from entity

`npx typeorm migration:generate UserMigration`

and to run the migration

`npx typeorm migration: run`


For me I did not call my migration from my lambda for various reasons.

I simple just changed the database configuration and ran the migration from my local pc.
# Deploy the project

`sls deploy --stage dev`

--stage dev: is optional, it just configures that stage variable is dev

# RDS secrets configuration on lambda

# Trying the lambda function

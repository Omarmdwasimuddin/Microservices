## Microservice

#### Package Installation
```bash
npm i --save @nestjs/microservices
```
> NestJS version 11.2.7 hole-->
```bash
npm i --save @nestjs/microservices@^11
```
---


#### `main.ts`
```bash
import { NestFactory } from '@nestjs/core';
import { Transport, MicroserviceOptions } from '@nestjs/microservices';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.createMicroservice<MicroserviceOptions>(
    AppModule,
    {
      transport: Transport.TCP,
      options: {
        host: '127.0.0.1',
        port: 4001,
      },
    },
  );

  await app.listen();
}
bootstrap();
```
---


#### `app.controller.ts`
```bash
import { Controller, Get } from '@nestjs/common';
import { MessagePattern } from '@nestjs/microservices';


@Controller()
export class AppController {
  @MessagePattern({ cmd: 'sum' })
  accumulate(data: number[]): number {
    return data.reduce((total, value) => total + value, 0);
  }
}
```
---


#### project run korle error na ashle sob thik ache.
```bash
npm run start:dev
```
<img width="851" height="201" alt="image" src="https://github.com/user-attachments/assets/9573daf6-04d3-4bf3-99d6-8ec74d2e7799" />



## TCP Microservice Client Setup

#### `app.module.ts`
```bash
import { Module } from '@nestjs/common';
import { ClientsModule, Transport } from '@nestjs/microservices';
import { AppController } from './app.controller';

@Module({
  imports: [
    ClientsModule.register([
      {
        name: 'MATH_SERVICE',
        transport: Transport.TCP,
        options: {
          host: '127.0.0.1',
          port: 4001,
        },
      },
    ]),
  ],
  controllers: [AppController],
})
export class AppModule {}
```
---


#### `app.controller.ts`
```bash
import { Controller, Get, Inject } from '@nestjs/common';
import { ClientProxy } from '@nestjs/microservices';

@Controller()
export class AppController {
  constructor(
    @Inject('MATH_SERVICE')
    private readonly mathClient: ClientProxy,
  ) {}

  @Get('sum')
  sum() {
    return this.mathClient.send<number>(
      { cmd: 'sum' },
      [10, 20, 30],
    );
  }
}
```
---

#### project run koro
```bash
npm run start:dev
```
> browser/Postman hit koro
```bash
http://localhost:3000/sum
```
---

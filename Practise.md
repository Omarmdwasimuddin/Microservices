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



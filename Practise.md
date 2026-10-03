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
<img width="851" height="201" alt="image" src="https://github.com/user-attachments/assets/9573daf6-04d3-4bf3-99d6-8ec74d2e7799" />

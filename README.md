# Microservices — Overview (NestJS)

Nest এ বেশ কিছু built-in transport layer implementation আছে, যেগুলোকে **transporter** বলা হয় — এগুলো বিভিন্ন microservice instance এর মধ্যে message পাঠানোর কাজ করে। বেশিরভাগ transporter ই native ভাবে request-response আর event-based — দুই ধরনের message style ই support করে। Nest প্রতিটা transporter এর implementation detail কে একটা canonical interface এর পেছনে abstract করে রাখে, দুই style এর জন্যই। এর ফলে, application code পরিবর্তন না করেই এক transport layer থেকে আরেকটায় switch করা যায় (যেমন, কোনো নির্দিষ্ট transport layer এর reliability বা performance feature এর সুবিধা নেওয়ার জন্য)।

---

## 1. Installation

Microservice বানানো শুরু করার আগে, প্রথমে প্রয়োজনীয় package টা install করো:

```bash
npm i --save @nestjs/microservices
```

---

## 2. শুরু করা

একটা microservice instantiate করার জন্য, `NestFactory` class এর `createMicroservice()` method ব্যবহার করো:

```typescript
// main.ts
import { NestFactory } from '@nestjs/core';
import { Transport, MicroserviceOptions } from '@nestjs/microservices';
import { AppModule } from './app.module.js';

async function bootstrap() {
  const app = await NestFactory.createMicroservice<MicroserviceOptions>(
    AppModule,
    {
      transport: Transport.TCP,
    },
  );
  await app.listen();
}
await bootstrap();
```

> **Hint:** Microservice গুলো default ভাবে TCP transport layer ব্যবহার করে।

`createMicroservice()` method এর দ্বিতীয় argument হলো একটা options object, যার দুটো member আছে:

| Property | বিবরণ |
|---|---|
| `transport` | কোন transporter ব্যবহার হবে সেটা specify করে (যেমন, `Transport.NATS`) |
| `options` | Transporter-specific একটা options object, যেটা transporter এর behavior ঠিক করে |

Options object টা বেছে নেওয়া transporter অনুযায়ী নির্দিষ্ট। TCP transporter নিচের property গুলো expose করে। অন্য transporter (যেমন Redis বা MQTT) এর available option জানতে, সেই সংশ্লিষ্ট chapter দেখো।

| Property | বিবরণ |
|---|---|
| `host` | Connection hostname |
| `port` | Connection port |
| `retryAttempts` | Unexpectedly close হয়ে যাওয়ার পর server কতবার আবার listen করার চেষ্টা করবে (default: 0) |
| `retryDelay` | ওই attempt গুলোর মাঝে delay (ms) (default: 0) |
| `serializer` | Outgoing message এর জন্য custom serializer |
| `deserializer` | Incoming message এর জন্য custom deserializer |
| `socketClass` | `TcpSocket` extend করা একটা custom Socket (default: `JsonSocket`) |
| `tlsOptions` | TLS protocol configure করার option (দেখো [TLS support](#8-tls-support)) |
| `maxBufferSize` | Incoming message এর জন্য buffer এর সর্বোচ্চ size, characters এ (default: `(512 * 1024 * 1024) / 4`) |
| `incompleteMessageTimeout` | একটা packet এর মাঝখানে একটা peer কতক্ষণ (ms) silent থাকলে connection drop হয়ে যাবে। Disable করতে `0` সেট করো (default: 30000) |
| `maxSendBufferSize` | কোনো peer response গুলো না পড়লে, তার জন্য সর্বোচ্চ কতটা response bytes queue হতে পারে, তারপর connection drop হয়ে যাবে। Disable করতে `0` সেট করো (default: 128MB) |

---

## 3. Message এবং Event Pattern

Microservice, message আর event — দুটোই pattern দিয়ে চেনে। একটা pattern হলো একটা plain value — যেমন, একটা literal object বা একটা string। Pattern গুলো automatically serialize হয়ে message এর data portion এর সাথে network এ পাঠানো হয়। এভাবে, message sender আর consumer রা কোন request কোন handler consume করবে, সেটা নিয়ে coordinate করতে পারে।

### 3.1 Request-Response

Service গুলোর মধ্যে message exchange করতে হলে request-response message style টা useful। এটা নিশ্চিত করে যে service টা আসলেই message পেয়েছে, তোমাকে manually কোনো acknowledgment protocol implement করতে হবে না। তবে, request-response সবসময় সবচেয়ে ভালো fit না। যেমন, Kafka বা NATS JetStream এর মতো log-based persistence ব্যবহার করা streaming platform গুলো একেবারে ভিন্ন ধরনের challenge এর জন্য optimize করা, যেগুলো event messaging paradigm এর সাথে বেশি মেলে (বিস্তারিত জানতে event-based messaging দেখো)।

Request-response message type enable করতে, Nest দুটো logical channel তৈরি করে: একটা data transfer করার জন্য, আরেকটা incoming response এর জন্য অপেক্ষা করার জন্য। NATS এর মতো কিছু underlying transport এ, এই dual-channel support built-in ভাবেই পাওয়া যায়। অন্যদের ক্ষেত্রে, Nest manually আলাদা channel তৈরি করে এটা compensate করে, যেটা কিছুটা overhead তৈরি করতে পারে। যদি request-response message style না লাগে, তাহলে event-based method ব্যবহার করাই ভালো।

Request-response paradigm এর উপর ভিত্তি করে একটা message handler বানাতে, `@nestjs/microservices` package থেকে import করা `@MessagePattern()` decorator ব্যবহার করো। এই decorator শুধু controller class এর ভেতরে ব্যবহার করো, কারণ এগুলোই তোমার application এর entry point হিসেবে কাজ করে। Provider এর ভেতরে এটা ব্যবহার করলে Nest runtime সেটা ignore করে।

```typescript
// math.controller.ts
import { Controller } from '@nestjs/common';
import { MessagePattern } from '@nestjs/microservices';

@Controller()
export class MathController {
  @MessagePattern({ cmd: 'sum' })
  accumulate(data: number[]): number {
    return (data || []).reduce((a, b) => a + b);
  }
}
```

উপরের code এ, `accumulate()` message handler টা `{ cmd: 'sum' }` message pattern এর সাথে মিলে যাওয়া message গুলোর জন্য listen করে। এই message handler একটা single argument নেয় — client থেকে পাঠানো data। এই ক্ষেত্রে, data হলো একটা number এর array, যেগুলো accumulate করতে হবে।

#### Asynchronous Response

Message handler গুলো synchronously বা asynchronously — দুইভাবেই response দিতে পারে, তাই async method ও support করে।

```typescript
@MessagePattern({ cmd: 'sum' })
async accumulate(data: number[]): Promise<number> {
  return (data || []).reduce((a, b) => a + b);
}
```

একটা message handler একটা `Observable` ও return করতে পারে, সেক্ষেত্রে stream শেষ না হওয়া পর্যন্ত result value গুলো emit হতে থাকে।

```typescript
@MessagePattern({ cmd: 'sum' })
accumulate(data: number[]): Observable<number> {
  return from([1, 2, 3]);
}
```

উপরের example এ, message handler টা তিনবার response দেয় — array এর প্রতিটা item এর জন্য একবার করে।

### 3.2 Event-Based

Service গুলোর মধ্যে message exchange করার জন্য request-response method ভালো কাজ করলেও, event-based messaging এর জন্য এটা ততটা suitable না — যেখানে তুমি response এর জন্য অপেক্ষা না করেই event publish করতে চাও। এই ধরনের ক্ষেত্রে, request-response এর জন্য দুটো channel maintain করার overhead টা অপ্রয়োজনীয়।

যেমন, system এর এই অংশে একটা নির্দিষ্ট condition ঘটেছে সেটা অন্য একটা service কে জানাতে চাইলে, event-based message style ব্যবহার করো।

একটা event handler বানাতে, `@nestjs/microservices` package থেকে import করা `@EventPattern()` decorator ব্যবহার করো।

```typescript
@EventPattern('user_created')
async handleUserCreated(data: Record<string, unknown>) {
  // business logic
}
```

> **Hint:** একটা single event pattern এর জন্য একাধিক event handler register করা যায়, আর Nest এগুলো সব একসাথে (parallel এ) trigger করে।

`handleUserCreated()` event handler টা `'user_created'` event এর জন্য listen করে। এই event handler একটা single argument নেয় — client থেকে পাঠানো data (এই ক্ষেত্রে, network এ পাঠানো একটা event payload)।

---

## 4. Request সম্পর্কে অতিরিক্ত তথ্য

আরো advanced scenario তে, incoming request সম্পর্কে অতিরিক্ত detail প্রয়োজন হতে পারে। যেমন, wildcard subscription সহ NATS ব্যবহার করার সময়, producer যেই original subject এ message পাঠিয়েছে সেটা তুমি retrieve করতে চাইতে পারো। একইভাবে, Kafka এর ক্ষেত্রে, message header access করার প্রয়োজন হতে পারে। এর জন্য নিচের built-in decorator গুলো ব্যবহার করো:

```typescript
@MessagePattern('time.us.*')
getDate(@Payload() data: number[], @Ctx() context: NatsContext) {
  console.log(`Subject: ${context.getSubject()}`); // e.g. "time.us.east"
  return new Date().toLocaleTimeString(...);
}
```

> **Hint:** `@Payload()`, `@Ctx()`, আর `NatsContext` — এগুলো `@nestjs/microservices` থেকে import করা হয়।

> **Hint:** Incoming payload object থেকে একটা নির্দিষ্ট property বের করার জন্য `@Payload()` decorator এ একটা property key পাস করা যায়, যেমন `@Payload('id')`। Payload কে একটা schema এর বিপরীতে validate করার জন্য, microservice pipes দেখো।

---

## 5. Client (Producer Class)

একটা client Nest application `ClientProxy` class ব্যবহার করে একটা Nest microservice এর সাথে message exchange করতে পারে, অথবা তাতে event publish করতে পারে। এই class একটা remote microservice এর সাথে communicate করার বেশ কিছু method দেয়, যেমন `send()` (request-response messaging এর জন্য) আর `emit()` (event-driven messaging এর জন্য)। এই class এর একটা instance নিচের উপায় গুলোতে পাওয়া যায়।

একটা approach হলো `ClientsModule` import করা, যেটা static `register()` method expose করে। এই method microservice transporter represent করা object গুলোর একটা array নেয়। প্রতিটা object এ একটা `name` property থাকতেই হবে, আর একটা `transport` property (না দিলে, Nest `Transport.TCP` ব্যবহার করে) এবং একটা `options` property থাকতে পারে।

`name` property একটা injection token হিসেবে কাজ করে, যেটা তুমি যেখানেই দরকার সেখানে `ClientProxy` এর একটা instance inject করতে ব্যবহার করতে পারো। এর value যেকোনো string বা JavaScript symbol হতে পারে, non-class-based provider token এ যেমন বলা আছে।

`options` property হলো একটা object, যার property গুলো আগে `createMicroservice()` method এ যেভাবে দেখেছি সেরকমই।

```typescript
@Module({
  imports: [
    ClientsModule.register([
      { name: 'MATH_SERVICE', transport: Transport.TCP },
    ]),
  ],
})
```

বিকল্পভাবে, setup এর সময় যদি configuration দিতে হয় বা অন্য কোনো asynchronous process করতে হয়, তাহলে `registerAsync()` method ব্যবহার করো।

```typescript
@Module({
  imports: [
    ClientsModule.registerAsync([
      {
        imports: [ConfigModule],
        name: 'MATH_SERVICE',
        useFactory: async (configService: ConfigService) => ({
          transport: Transport.TCP,
          options: {
            host: configService.get('HOST'),
            port: configService.get('PORT'),
          },
        }),
        inject: [ConfigService],
      },
    ]),
  ],
})
```

Module import হয়ে গেলে, `'MATH_SERVICE'` transporter এর জন্য configure করা `ClientProxy` instance inject করতে `@Inject()` decorator ব্যবহার করো।

```typescript
constructor(
  @Inject('MATH_SERVICE') private client: ClientProxy,
) {}
```

> **Hint:** `ClientsModule` আর `ClientProxy` class দুটো `@nestjs/microservices` package থেকে import করা হয়।

কখনো কখনো, transporter এর configuration client application এ hard-code করার বদলে অন্য কোনো service (যেমন `ConfigService`) থেকে fetch করার প্রয়োজন হতে পারে। এর জন্য, `ClientProxyFactory` class ব্যবহার করে একটা custom provider register করো। এই class একটা static `create()` method দেয়, যেটা একটা transporter options object নেয় এবং একটা customized `ClientProxy` instance return করে।

```typescript
@Module({
  providers: [
    {
      provide: 'MATH_SERVICE',
      useFactory: (configService: ConfigService) => {
        const mathSvcOptions = configService.getMathSvcOptions();
        return ClientProxyFactory.create(mathSvcOptions);
      },
      inject: [ConfigService],
    },
  ],
  // ...
})
```

> **Hint:** `ClientProxyFactory` class টা `@nestjs/microservices` package থেকে import করা হয়।

আরেকটা option হলো `@Client()` property decorator ব্যবহার করা।

```typescript
@Client({ transport: Transport.TCP })
client: ClientProxy;
```

> **Hint:** `@Client()` decorator টা `@nestjs/microservices` package থেকে import করা হয়।

`@Client()` decorator টা preferred technique না, কারণ এভাবে তৈরি করা একটা client test করা এবং share করা তুলনামূলক কঠিন।

`ClientProxy` হলো **lazy**। এটা সাথে সাথে কোনো connection initiate করে না। বরং, প্রথম microservice call এর আগে connection তৈরি হয় এবং পরবর্তী প্রতিটা call এ সেটাই reuse হয়। যদি একটা connection তৈরি না হওয়া পর্যন্ত application bootstrapping process delay করতে চাও, তাহলে `onApplicationBootstrap()` lifecycle hook এর ভেতরে `ClientProxy` object এর `connect()` method দিয়ে manually connection initiate করো।

```typescript
async onApplicationBootstrap() {
  await this.client.connect();
}
```

যদি connection তৈরি করা না যায়, তাহলে `connect()` method সংশ্লিষ্ট error object সহ reject করে।

### 5.1 Message পাঠানো

`ClientProxy` একটা `send()` method expose করে, যেটা microservice কে call করে এবং তার response সহ একটা `Observable` return করে।

```typescript
accumulate(): Observable<number> {
  const pattern = { cmd: 'sum' };
  const payload = [1, 2, 3];
  return this.client.send<number>(pattern, payload);
}
```

`send()` method দুটো argument নেয় — `pattern` আর `payload`। `pattern` টা কোনো `@MessagePattern()` decorator এ define করা একটা pattern এর সাথে match করা উচিত। `payload` হলো remote microservice এ পাঠানোর message। এই method একটা cold `Observable` return করে, মানে হলো, message পাঠানোর আগে তোমাকে explicitly সেটাতে subscribe করতে হবে।

### 5.2 Event Publish করা

একটা event পাঠানোর জন্য, `ClientProxy` object এর `emit()` method ব্যবহার করো। এই method message broker এ একটা event publish করে।

```typescript
async publish() {
  this.client.emit<number>('user_created', new UserCreatedEvent());
}
```

`emit()` method দুটো argument নেয় — `pattern` আর `payload`। `pattern` টা কোনো `@EventPattern()` decorator এ define করা একটা pattern এর সাথে match করা উচিত, আর `payload` হলো remote microservice এ পাঠানোর event data। এই method একটা hot `Observable` return করে (`send()` যেই cold `Observable` return করে তার বিপরীত), মানে হলো, proxy সাথে সাথেই event টা deliver করার চেষ্টা করে, তুমি observable তে subscribe করো বা না করো।

---

## 6. Request-Scoping

তুমি যদি অন্য কোনো programming language থেকে এসে থাকো, তাহলে এটা জেনে অবাক হতে পারো যে Nest এ, বেশিরভাগ জিনিস ই incoming request গুলোর মধ্যে shared থাকে। এর মধ্যে আছে database connection pool, global state সহ singleton service, আর আরো অনেক কিছু। Node.js request/response multi-threaded stateless model follow করে না, যেখানে প্রতিটা request আলাদা একটা thread এ process হয়। ফলে, singleton instance ব্যবহার করা তোমার application এর জন্য safe।

তবে, কিছু edge case আছে যেখানে handler এর জন্য একটা request-based lifetime দরকার হতে পারে — যেমন GraphQL application এ per-request caching, request tracking, বা multi-tenancy। Scope কীভাবে control করবে সেটা জানতে injection scopes chapter দেখো।

Request-scoped handler আর provider গুলো `@Inject()` decorator আর `CONTEXT` token একসাথে ব্যবহার করে `RequestContext` inject করতে পারে:

```typescript
import { Injectable, Scope, Inject } from '@nestjs/common';
import { CONTEXT, RequestContext } from '@nestjs/microservices';

@Injectable({ scope: Scope.REQUEST })
export class CatsService {
  constructor(@Inject(CONTEXT) private ctx: RequestContext) {}
}
```

এর মাধ্যমে `RequestContext` object access করা যায়, যেটার shape নিচের মতো:

```typescript
export interface RequestContext<TData = any, TContext extends BaseRpcContext = any> {
  pattern: string | Record<string, any>;
  data: TData;
  context?: TContext;
  getData(): TData;
  getPattern(): string | Record<string, any>;
  getContext(): TContext;
}
```

`data` property হলো message producer এর পাঠানো message payload। `pattern` property হলো incoming message এর জন্য handler চিহ্নিত করতে ব্যবহৃত pattern। `context` property এ থাকে transporter-specific context object (যেমন `NatsContext`) — একই object, যেটা `@Ctx()` decorator inject করে।

---

## 7. Instance Status Update

Connection এবং underlying driver instance এর state সম্পর্কে real-time update পেতে, status stream এ subscribe করো। এই stream বেছে নেওয়া driver অনুযায়ী নির্দিষ্ট status update দেয়। যেমন, TCP transporter (default) এর ক্ষেত্রে, status stream `connected` আর `disconnected` — এই দুই ধরনের event emit করে।

```typescript
this.client.status.subscribe((status: TcpStatus) => {
  console.log(status);
});
```

> **Hint:** `TcpStatus` type টা `@nestjs/microservices` package থেকে import করা হয়।

একইভাবে, server এর status সম্পর্কে notification পেতে, server এর status stream এও subscribe করা যায়।

```typescript
const server = app.connectMicroservice<MicroserviceOptions>(...);
server.status.subscribe((status: TcpStatus) => {
  console.log(status);
});
```

### 7.1 Internal Event Listen করা

কিছু ক্ষেত্রে, microservice এর emit করা internal event গুলো listen করার দরকার হতে পারে। যেমন, কোনো error ঘটলে অতিরিক্ত operation trigger করার জন্য `error` event এর জন্য listen করতে পারো। এর জন্য, `on()` method ব্যবহার করো:

```typescript
this.client.on('error', (err) => {
  console.error(err);
});
```

একইভাবে, server এর internal event ও listen করা যায়:

```typescript
server.on<TcpEvents>('error', (err) => {
  console.error(err);
});
```

> **Hint:** `TcpEvents` type টা `@nestjs/microservices` package থেকে import করা হয়।

### 7.2 Underlying Driver Access

আরো advanced use case এ, underlying driver instance access করার দরকার হতে পারে — যেমন, manually connection close করা বা driver-specific method ব্যবহার করা। তবে, বেশিরভাগ ক্ষেত্রেই driver directly access করার দরকার হয় না।

এর জন্য, `unwrap()` method ব্যবহার করো, যেটা underlying driver instance return করে। Generic type parameter দিয়ে বলা যায় তুমি কোন type এর driver instance আশা করছো।

```typescript
const netServer = this.client.unwrap<Server>();
```

এখানে, `Server` হলো `net` module থেকে import করা একটা type।

একইভাবে, server এর underlying driver instance ও access করা যায়:

```typescript
const netServer = server.unwrap<Server>();
```

---

## 8. Timeout Handle করা

Distributed system এ, মাঝেমধ্যে microservice down থাকে বা unavailable হয়ে যায়। অনির্দিষ্টকালের জন্য অপেক্ষা না করার জন্য, RxJS এর `timeout` operator দিয়ে microservice call এ একটা timeout apply করো। নির্দিষ্ট সময়ের মধ্যে microservice response না দিলে, একটা error throw হয়, যেটা তুমি catch করে যথাযথভাবে handle করতে পারো।

`pipe` এর ভেতরে `timeout` operator apply করো:

```typescript
this.client
  .send<TResult, TInput>(pattern, data)
  .pipe(timeout(5000));
```

> **Hint:** `timeout` operator টা `rxjs/operators` package থেকে import করা হয়।

Microservice ৫ সেকেন্ডের মধ্যে response না দিলে, `Observable` একটা `TimeoutError` সহ error করে।

---

## 9. Service জুড়ে একটা Request Trace করা

Timeout শুধু বলে দেয় যে একটা call সময়মতো ফেরত আসেনি। কিন্তু কোথায় সময়টা গেল, সেটা বলে না — আর TCP, NATS, আর Kafka এর মধ্যে দিয়ে কথা বলা পাঁচটা service এর system এ, এটাই আসল প্রশ্ন। একটা gateway request যেটা ৩ সেকেন্ড লাগায়, তার মধ্যে ২.৯ সেকেন্ড হয়তো কোনো downstream service এ ব্যয় হয়েছে যেটার উপর কারো সন্দেহই ছিল না, অথচ প্রতিটা service এর নিজের log দেখাচ্ছে সবকিছু ঠিক আছে।

সাধারণ সমাধান হলো, প্রতিটা transport এর মধ্য দিয়ে manually একটা correlation ID propagate করা, আর পরে সেই timeline গুলো একসাথে জোড়া দেওয়া। NestJS Observe এই জোড়া দেওয়ার কাজটা তোমার জন্য করে দেয়। প্রতিটা service কে `@nestjs/observe` SDK দিয়ে instrument করো, এবং transport এ যেই channel আগে থেকেই খালি আছে (একটা Kafka header, একটা NATS header, অথবা TCP payload এ একটা field) সেখানে trace ID forward করো — তাহলে dashboard এ, অংশ নেওয়া প্রতিটা service জুড়ে একটাই waterfall পুনর্গঠিত হয়ে যায়।

এরপর, timeout টা আর রহস্য থাকে না। তুমি দেখতে পাবে gateway এর `send()` অপেক্ষা করছে, consumer message টা তুলছে (আর তার আগে message টা কতক্ষণ অপেক্ষা করেছিল), handler এর ভেতরে যেই query টা বেশি সময় নিয়েছে, আর শেষে সেই error যেটা throw হয়েছে — সবকিছু, source line সহ, একটাই clock এ। Message আর event handler গুলো automatically instrument হয়ে যায়, তাই কোনো manual span wiring ছাড়াই `@MessagePattern()` আর `@EventPattern()` handler গুলো operation হিসেবে দেখা যায়।

Trace ID forward করাটাই একমাত্র অংশ যেটার জন্য application code লাগে, কারণ তোমার transport কোন channel টা খালি রেখেছে সেটা শুধু তুমিই জানো। প্রতিটা transport এর জন্য pattern জানতে Distributed tracing দেখো, আর setup করার জন্য Observability chapter দেখো।

---

## 10. TLS Support

Private network এর বাইরে communicate করার সময়, traffic টা encrypt করা উচিত। TCP transporter এ Node এর TLS module এর উপর ভিত্তি করে built-in TLS support আছে, যেটা দিয়ে microservice গুলো আর তাদের client এর মধ্যে communication encrypt করা যায়।

একটা TCP server এর জন্য TLS enable করতে, তোমার একটা private key আর একটা certificate লাগবে, PEM format এ। এগুলো `tlsOptions` property দিয়ে server এর options এ add করো:

```typescript
import * as fs from 'node:fs';
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module.js';
import { MicroserviceOptions, Transport } from '@nestjs/microservices';

async function bootstrap() {
  const key = fs.readFileSync('<pathToKeyFile>', 'utf8').toString();
  const cert = fs.readFileSync('<pathToCertFile>', 'utf8').toString();

  const app = await NestFactory.createMicroservice<MicroserviceOptions>(
    AppModule,
    {
      transport: Transport.TCP,
      options: {
        tlsOptions: {
          key,
          cert,
        },
      },
    },
  );

  await app.listen();
}
await bootstrap();
```

একটা client কে TLS এর মাধ্যমে securely communicate করানোর জন্য, `tlsOptions` object টা client এও define করতে হবে, এবার CA certificate সহ — অর্থাৎ, যেই authority server এর certificate টা sign করেছে, তার certificate। এটা নিশ্চিত করে যে client, server এর certificate কে trust করে এবং একটা secure connection establish করতে পারে।

```typescript
import * as fs from 'node:fs';
import { Module } from '@nestjs/common';
import { ClientsModule, Transport } from '@nestjs/microservices';

@Module({
  imports: [
    ClientsModule.register([
      {
        name: 'MATH_SERVICE',
        transport: Transport.TCP,
        options: {
          tlsOptions: {
            ca: [fs.readFileSync('<pathToCaFile>', 'utf-8').toString()],
          },
        },
      },
    ]),
  ],
})
export class AppModule {}
```

`ca` property একটা array নেয়, তাই যদি তোমার setup এ একাধিক trusted authority থাকে, তাহলে একাধিক CA list করতে পারবে।

সব কিছু setup হয়ে গেলে, আগের মতোই `@Inject()` decorator দিয়ে `ClientProxy` inject করো। তোমার microservice গুলোর মধ্যে communication তখন encrypted হয়ে যায়, encryption এর detail Node এর TLS module handle করে।

আরো তথ্যের জন্য, Node এর TLS documentation দেখো।

---

## 11. Dynamic Configuration

একটা microservice এর transport option গুলো `createMicroservice()` এ পাস করা হয়, কোনো provider (যেমন `@nestjs/config` package এর `ConfigService`) inject হওয়ার আগেই। Inject করা provider দিয়ে microservice configure করতে চাইলে, তার বদলে `AsyncMicroserviceOptions` পাস করো: Nest, `inject` এ list করা provider গুলো resolve করে সেগুলো `useFactory` function এ পাস করে, যেটা transport options return করে।

```typescript
import { NestFactory } from '@nestjs/core';
import { ConfigService } from '@nestjs/config';
import { AsyncMicroserviceOptions, Transport } from '@nestjs/microservices';
import { AppModule } from './app.module.js';

async function bootstrap() {
  const app = await NestFactory.createMicroservice<AsyncMicroserviceOptions>(
    AppModule,
    {
      useFactory: (configService: ConfigService) => ({
        transport: Transport.TCP,
        options: {
          host: configService.get<string>('HOST'),
          port: configService.get<number>('PORT'),
        },
      }),
      inject: [ConfigService],
    },
  );

  await app.listen();
}
await bootstrap();
```

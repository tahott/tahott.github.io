+++
title = "카페 주문 시스템 #1"
date = "2022-03-22"
created = "2022-03-22"
[taxonomies]
tags = ["NestJs", "monorepo"]
+++

#### day 1
고래는 개발자 시절 모놀리틱 레포 구조의 경험만 있었다. 하지만 전날 스케치한 아키텍처는 비동기 메시지 기반 통신을 지향하고 주문/결제/바리스타의 영역에서 한 쪽의 장애가 다른 영역에 영향을 끼치게 하고 싶지 않았다. 그래서 만들어둔 프로젝트 구조를 조금 변경 할 필요를 느꼈다. 다행히 NestJS는 <a href="https://docs.nestjs.com/cli/monorepo">모노레포</a>로 workspace를 만들 수 있는 환경을 지원하고 있어 적용해보기로 하였다.
```zsh
root $ nest generate app order
```
```bash
.
|-- apps/
  |-- order/
    |-- src/
    |-- test/
    |-- tsconfig.app.json
  |-- [project-name]/
|-- dist/
|-- docker/
  |-- docker-compose.yml
|-- node_modules/
|-- .eslintrc.js
|-- .gitignore
|-- .prettierrc
|-- nest-cli.json
|-- package.json
|-- README.md
|-- tsconfig.build.json
|-- tsconfig.json
|-- yarn.lock
```
nest 명령어를 통해 프로젝트 구조가 변경 되었고 기존의 `proeject-name`으로 되어있던 폴더와 nest-cli.json에 있는 `project-name` 정보를 함께 삭제하기로 했다.
그리고 로컬 개발 환경에서 카프카를 활용하기 위해 아래 명령어로 docker 폴더 안에 docker-compose 파일을 다운 받자
```bash
root/docker $ curl --silent --output docker-compose.yml \
  https://raw.githubusercontent.com/confluentinc/cp-all-in-one/7.0.1-post/cp-all-in-one/docker-compose.yml
```
```bash
root $ yarn add kafkajs @nestjs/microservices
```
NestJS에서 카프카를 사용하기 위해 kafkajs, @nestjs/microservices를 추가하며 시작할 준비가 끝난 듯 하다.<br>
언제나 세팅 과정은 힘들다. 여기까지 했으면 스트레칭도 하고 커피도 마시며 쉬고 오자 :coffee:

```ts
// main.ts
import { NestFactory } from '@nestjs/core';
import { MicroserviceOptions, Transport } from '@nestjs/microservices';
import { OrderModule } from './order.module';

async function bootstrap() {
  const app = await NestFactory.create(OrderModule);

  app.connectMicroservice<MicroserviceOptions>({
    transport: Transport.KAFKA,
    options: {
      client: {
        clientId: 'my-cafe',
        brokers: ['localhost:9092'],
      },
      consumer: {
        groupId: 'cafe-order',
      },
    },
  });

  await app.listen(3000);
}
bootstrap();
```
```ts
// order.module.ts
imports: [
  ClientsModule.register([
    {
      name: 'MY-CAFE-ORDER',
      transport: Transport.KAFKA,
      options: {
        client: {
          clientId: 'my-cafe',
          brokers: ['localhost:9092'],
        },
        consumer: {
          groupId: 'cafe-order',
        },
      },
    }
  ]),
],
```
```ts
// order.controller.ts
export class OrderController implements OnModuleInit, OnModuleDestroy {
  constructor(
    private readonly orderService: OrderService,
    @Inject('MY-CAFE-ORDER') private readonly client: ClientKafka,
  ) { }
  async onModuleInit() {
    await this.client.connect();
  }

  async onModuleDestroy() {
    await this.client.close();
  }

  @Get()
  getHello() {
    return this.orderService.getHello();
  }

  @Post()
  order() {
    return this.client.emit('order', `order created... ${new Date()}`);
  }
}
```
고래는 NestJS의 공식문서를 살펴보며 위와 같은 코드를 짰다. 아래의 명령어로 order 앱과 카프카를 실행시킨 후 order api를 Post method로 호출하면 'order' 토픽에 메시지가 쌓이는 것을 기대 할 수 있다.

```bash
root/docker $ docker-compose up -d
root $ nest start order
```

이제 카프카의 토픽에 쌓인 메시지를 consumer하는 역의 앱을 추가하여 확인해 보자. 고객이 주문을 생성하면 결제정보가 함께 넘어오니
결제 정보를 가져와 승인을 받기 위한 `payment` 앱을 추가하기로 한다.
```bash
root $ nest g app payment
.
|-- apps/
  |-- order/
  |-- payment/
```

payment 앱의 main.ts, payment.module.ts도 order와 같이 카프카를 위한 설정을 추가해준 뒤 controller에서 메시지 수신을 하기로 했다
```ts
// payment.controller.ts
@MessagePattern('order')
  recvMsg(
    @Payload() message: any,
    @Ctx() context: KafkaContext,
  ) {
    const originMessage = context.getMessage();

    console.log(JSON.stringify(originMessage.value));
    // "order created... Tue Mar dd 2022 hh:mm:ss GMT+0900 (대한민국 표준시)"
  }
```
payment 앱도 실행 시킨 뒤, order의 api를 호출하니 payment에서 'order' 토픽에 쌓인 메시지를 가져오는 것을 확인 할 수 있었다.

이제 간단하게 앱끼리의 pub/sub을 확인한 후 이제 주문-결제-제작의 사이클을 위한 코드를 추가하기로 한다.
+++
title = "카페 주문 시스템 #2"
date = "2022-03-24"
created = "2022-03-23"
[taxonomies]
tags = ["NestJs", "monorepo"]
+++

#### day 2
고래는 머릿속의 주문 과정을 다시 한 번 정리해보기로 했다.
1. 고객은 키오스크 또는 앱으로 음료를 주문한다
2. 결제 정보가 담긴 주문이 서버로 요청 된다
3. Order App이 주문 정보를 DB에 저장하고 주문 생성 이벤트를 발행한다
4. 주문 생성 이벤트가 발행되면 Payment App이 토픽에서 이벤트를 확인하여 결제 정보로 벤더사에 승인을 요청한다
5. 승인 여부를 담은 결제처리 이벤트를 페이먼츠가 발행한다
6. Order와 Barista 결제처리 이벤트를 확인한다.<br>
  6-1. Order는 결제 처리 여부를 고객에게 리턴한다<br>
  6-2. Barista는 결제가 성공이라면 주문 리스트에 주문 정보를 업데이트하고 순서대로 제작에 들어간다
7. 음료 제작이 완료되면 주문한 고객에게 알린다 (앱 또는 주문 현황 스크린)

세세하지 않지만 이런 흐름대로 처리하면 될거라 생각을 마친 후 일단 개발에 들어갔다.

payment에서 order에 등록 된 토픽을 가져오는 것까지 확인했으니 이어서 결제 처리 완료를 진행 할 차례다.
```ts
// payment.controller.ts
@MessagePattern('order')
async recvMsg(
  @Payload() message: any,
  @Ctx() context: KafkaContext,
) {
  const originMessage = context.getMessage();

  const paymentResult = await this.thirdPartyPay();

  const res = Object.assign(originMessage.value, { paymentResult });

  console.log(res)

  this.client.emit('payment', res);

  return res;
}

private async thirdPartyPay() {
  return Math.floor(Math.random() * (11 - 1) + 1) < 9;
}
```
간편 결제로 많이 지원하는 xx페이의 API 문서를 보면 최종 승인을 위한 token 혹은 id 값을 보내 벤더사에 최종 결제처리를 요청 하는 것을 볼 수 있었다.
주문을 할 때 간편결제류로 여러 페이를 지원한다는 가정하에 최종 결제를 요청 할 'thirdPartyPay'라는 함수를 만들고 1/10 확률로 결제 실패를 할 거라 가정해둔다.
그리고 기존의 주문 정보와 결제처리 결과를 payment라는 토픽으로 발행하며 값은 값을 리턴값으로 사용하여 order 측에 결제 처리 결과를 동기적으로 알려준다.

```ts
// barista.controller.ts
@MessagePattern('payment')
  recvMsg(
    @Payload() message: any,
    @Ctx() context: KafkaContext,
  ) {
    const originMessage = context.getMessage();

    if (JSON.parse(JSON.stringify(originMessage.value)).paymentResult) {
      /**
       * 주문이 리스트에 등록되면 바리스타는 작업에 들어간다
       * 바리스타는 실제로는 사람이지만 이 시스템에서는 바리스타의 행동을
       * 정의하는 서비스를 따로 두어 바리스타의 행동을 나타내기로 한다
       */
      BaristaService.insertOrder(JSON.parse(JSON.stringify(originMessage.value)).name)
    }
  }
```
```ts
// barista.service.ts
export class BaristaService {
  static #list: Array<string> = [];
  static #isMaking = false;

  constructor() { }

  static insertOrder(menu: string) {
    this.#list.push(menu);
  }

  static shiftOrder() {
    return this.#list.shift();
  }

  static currentBaristaWorkState() {
    return this.#isMaking;
  }

  static getOrderList() {
    return this.#list;
  }

  static setWorkBarista() {
    this.#isMaking = !this.#isMaking;
  }
}
```
바리스타(고래)의 행동은 전역적으로 공유 된다. 주문은 한 번에 하나씩 처리할 수 있고 먼저 들어온 주문 먼저 처리를 한다.
결제처리가 성공이되면 주문 리스트에 주문이 등록 된다.

```bash
$ yarn add @nestjs/schedule
|-- apps/
  |-- .../
  |-- barista/
  |-- src/
    |-- tasks/
      |-- tasks.module.ts
      |-- tasks.service.ts
```
```ts
// tasks.service.ts
@Injectable()
export class TasksService {
  constructor() { }
  
  @Interval(3000)
  async handleInterval() {
    if (BaristaService.getOrderList().length > 0
      && !BaristaService.currentBaristaWorkState()
    ) {
      BaristaService.setWorkBarista();
      const menu = BaristaService.shiftOrder();

      switch (menu) {
        case 'americano':
          await this.delay(3000);
          break;
        case 'latte':
          await this.delay(5000);
          break;
        default:
          break;
      }

      BaristaService.setWorkBarista();
      console.log(
        `Order completed... ${menu}. ${new Date()}`
      )
    }
  }

  private delay(ms: number) {
    return new Promise(resolve => setTimeout(resolve, ms))
  }
}
```
바리스타는 정기적으로 새로 등록 된 주문 리스트를 확인한다. 아직 처리 안 한 주문이 남아있고 바리스타가 일 하지 않는 상태라면
먼저 등록 된 주문을 꺼내 제조에 들어간다. 아메리카노는 n초, 라떼는 n초의 걸려 작업이 완료된다. 지금은 로그로 작업 완료를 나타내지만
작업이 완료 되면 고객이 확인 할 수 있는 화면에 완료 상태를 푸시하는 작업을 추가 할 수 있을 것이다.

상세하게는 아니지만 여기까지 하나의 사이클을 확인 한 것 같다. 앞으로는 각 app들의 세부사항 구현에 들어갈 것이라 생각하며 하루를 마무리한다.
그리고 지금까지의 코드는 <a href="https://github.com/tahott/cafe-msa">이곳에</a> 있다.

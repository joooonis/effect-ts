### Generator 함수는 iterator를 생성하는 특별한 함수입니다.

```tsx
function* generatorFunction() {
  yield 1;
  yield 2;
}
```

- `function*`로 Generator 함수를 선언합니다.
- `yield` 키워드로 값(value)을 반환하고 실행을 중지합니다.
- `next()`로 다음 값을 얻을 수 있습니다.
- Generator는 일회성 iterator 생성하고 소진되면 재사용이 불가능 합니다.

\*Generator의 iterator는 반복문에서 접근이 가능합니다.

```tsx
function* generateAlphabet() {
  yield "a";
  yield "b";
  yield "c";
}

function* generateNumbers() {
  yield 1;
  yield 2;
  yield* generateAlphabet(); // 다른 generator에 위임
  yield 3;
}

const gen = generateNumbers();
for (const value of gen) {
  console.log(value); // 1, 2, a, b, c, 3
}
```

### Generator를 사용한 양방향 통신 예시:

```tsx
function* conversation() {
  const response1 = yield "첫 번째 질문";
  console.log("응답1:", response1);
  const response2 = yield "두 번째 질문";
  console.log("응답2:", response2);
}

const gen = conversation();
console.log(gen.next().value); // "첫 번째 질문"
console.log(gen.next("답변1").value); // 응답1: 답변1, "두 번째 질문"
console.log(gen.next("답변2").value); // 응답2: 답변2, undefined
```

`next()`에 전달하는 인자는 이전 `yield`의 반환값이 됩니다.

```tsx
function* calculator() {
  console.log("시작");

  // 첫번째 yield의 반환값이 a에 할당됨
  const a = yield "첫번째 숫자를 주세요";
  console.log("받은 첫번째 숫자:", a);

  // 두번째 yield의 반환값이 b에 할당됨
  const b = yield "두번째 숫자를 주세요";
  console.log("받은 두번째 숫자:", b);

  return a + b; // 최종 결과 반환
}

const calc = calculator();

console.log(calc.next()); // {value: '첫번째 숫자를 주세요', done: false}
// 여기서는 아직 아무 값도 할당되지 않음

console.log(calc.next(10)); // {value: '두번째 숫자를 주세요', done: false}
// 10이 첫번째 yield의 반환값이 되어 a에 할당됨

console.log(calc.next(20)); // {value: 30, done: true}
// 20이 두번째 yield의 반환값이 되어 b에 할당됨
```

- 첫 번째 `next()`는 인자를 전달해도 의미가 없습니다. 아직 yield가 실행되지 않았기 때문입니다.
- 각 `next()`에 전달된 값은 "이전" yield의 반환값이 됩니다:

### Effect.gen

Effect.gen utility를 사용하면 generator 문법을 가지고 비동기 작업을 동기적으로 작성할 수 있습니다.

```tsx
import { Effect } from "effect";

// Function to add a small service charge to a transaction amount
const addServiceCharge = (amount: number) => amount + 1;

// Function to apply a discount safely to a transaction amount
const applyDiscount = (
  total: number,
  discountRate: number
): Effect.Effect<number, Error> =>
  discountRate === 0
    ? Effect.fail(new Error("Discount rate cannot be zero"))
    : Effect.succeed(total - (total * discountRate) / 100);

// Simulated asynchronous task to fetch a transaction amount from a
// database
const fetchTransactionAmount = Effect.promise(() => Promise.resolve(100));

// Simulated asynchronous task to fetch a discount rate from a
// configuration file
const fetchDiscountRate = Effect.promise(() => Promise.resolve(5));

// Assembling the program using a generator function
const program = Effect.gen(function* () {
  // Retrieve the transaction amount
  const transactionAmount = yield* fetchTransactionAmount;

  // Retrieve the discount rate
  const discountRate = yield* fetchDiscountRate;

  // Calculate discounted amount
  const discountedAmount = yield* applyDiscount(
    transactionAmount,
    discountRate
  );

  // Apply service charge
  const finalAmount = addServiceCharge(discountedAmount);

  // Return the total amount after applying the charge
  return `Final amount to charge: ${finalAmount}`;
});

// Execute the program and log the result
Effect.runPromise(program).then(console.log);
// Output: Final amount to charge: 96
```

```tsx
Effect.gen(function* () {
  // generator 함수 내부
});
```

여기서 `function*` 구문은 이 함수가 generator 함수임을 나타냅니다. generator 함수는 실행을 일시 중지하고 나중에 재개할 수 있는 특별한 함수입니다.

```tsx
const transactionAmount = yield * fetchTransactionAmount;
const discountRate = yield * fetchDiscountRate;
const discountedAmount = yield * applyDiscount(transactionAmount, discountRate);
```

- `yield*` 연산자는 다른 generator나 이터러블을 위임하는데 사용됩니다
- 이 코드에서는 비동기 작업들(Effect)을 순차적으로 실행하면서 각 작업의 결과를 기다리는데 사용됩니다
- 각 `yield*` 구문은 해당 Effect가 완료될 때까지 실행을 일시 중지합니다

이점 :

- generator를 사용함으로써 비동기 코드를 마치 동기 코드처럼 작성할 수 있습니다
- Promise 체이닝이나 콜백 지옥을 피하고 코드를 위에서 아래로 읽을 수 있게 됩니다
- 이러한 generator 패턴은 Effect 타입의 연산들을 순차적으로 실행하면서도 가독성 높은 코드를 작성할 수 있게 해주며, 복잡한 비동기 로직을 단순화하는데 도움을 줍니다.

\*Effect.gen은 실행중 첫 에러를 만나는 순간 중지되어 에러를 단언적으로 처리 할 수 있습니다.

```tsx
import { Effect, Console } from "effect";

const task1 = Console.log("task1...");
const task2 = Console.log("task2...");
const failure = Effect.fail("Something went wrong!");
const task4 = Console.log("task4...");

const program = Effect.gen(function* () {
  yield* task1;
  yield* task2;
  // The program stops here due to the error
  yield* failure;
  // The following lines never run
  yield* task4;
  return "some result";
});

Effect.runPromise(program).then(console.log, console.error);
/*
Output:
task1...
task2...
(FiberFailure) Error: Something went wrong!
*/
```

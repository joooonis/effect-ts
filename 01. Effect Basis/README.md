## Effect Basis

### What is Effect?

```
     ┌─── Represents the success type
     │        ┌─── Represents the error type
     │        │      ┌─── Represents required dependencies
     ▼        ▼      ▼
Effect<Success, Error, Requirements>
```

Effect는 그 자체로 함수와 유사하지만 그자체로는 실행되지 않고 그러한 작업을 위한 모델(선언)입니다. Effect는 `Effect Runtime System`에 의해 실행됩니다.

### Expected Error Handling

```tsx
// Type signature doesn't show possible exceptions
const divide = (a: number, b: number): number => {
  if (b === 0) {
    throw new Error("Cannot divide by zero");
  }
  return a / b;
};
```

전통적인 프로그래밍 방법처럼 함수가 Error를 throw하면 Type signature를 통해 Erro에 대한 타입을 추론하는 것은 불가능합니다.

Effect에서는 `Effect.succeed` `Effect.fail` 와 같은 생성자(Constructor)를 통해 성공 실패시의 Effect(효과) 모두를 명시적으로 제어할수 있게 해줍니다.

Example: Using “tagged” errors 에러 구분을 위해 \_tag를 사용

```tsx
import { Effect } from "effect";

class HttpError {
  readonly _tag = "HttpError";
}

//      ┌─── Effect<never, HttpError, never>
//      ▼
const program = Effect.fail(new HttpError());
```

### Thunk

Javascript에서 thunk는 동기(Synchronous)적인 계산을 즉시 실행하지 않고 나중에 실행할 수 있도록 감싸는 함수입니다.

```jsx
// 일반적인 동기 함수
const add = (x, y) => x + y;
console.log(add(2, 3)); // 즉시 실행되어 5 출력// thunk를 사용한 버전
const addThunk = (x, y) => () => x + y;
const calculation = addThunk(2, 3); // 아직 실행되지 않음
console.log(calculation()); // 나중에 실행되어 5 출력
```

Example: Effect With Thunk

```tsx
import { Effect } from "effect";

const log = (message: string) =>
  Effect.sync(() => {
    console.log(message); // side effect
  });

//      ┌─── Effect<void, never, never>
//      ▼
const program = log("Hello, World!");
```

### Synchronous Effect

Effect.sync(항상 성공을 보장), Effect.try(실패할 수 있음)를 사용하여 thunk를 활용하는 synchronous side effects를 다룰수 있습니다.

```tsx
import { Effect } from "effect";

const log = (message: string) =>
  Effect.sync(() => {
    console.log(message); // side effect
  });

//      ┌─── Effect<void, never, never>
//      ▼
const program = log("Hello, World!");
```

### **Asynchronous Effects**

Javascript의 `Promise<Value>`는 성공 케이스의 타입만 표현할 수 있고 이는 에러 타입이 타입 시스템에 반영되지 않아 에러 처리가 제한적입니디. Effect.promise(항상 성공을 보장)와 Effect.tryPromise(실패할 수 있음) 두 가지 생성자를 제공히고 비동기 컨텍스트에서 성공과 실패 두 케이스를 모두 명시적으로 다룰 수 있습니다

Example: Customizing Error Handling

```tsx
import { Effect } from "effect";

const getTodo = (id: number) =>
  Effect.tryPromise({
    try: () => fetch(`https://jsonplaceholder.typicode.com/todos/${id}`),
    // remap the error
    catch: (unknown) => new Error(`something went wrong ${unknown}`),
  });

//      ┌─── Effect<Response, Error, never>
//      ▼
const program = getTodo(1);
```

Effect.async는 콜백 기반 API를 Effect로 변환할 때 사용합니다.(예를들면 NodeFS.readFile 파일 읽기 등) 이때 내부에서 resume은 한 번만 호출되어야 합니다.

```tsx
// 기본 구조
Effect.async<성공타입, 에러타입>((resume) => {
  // 비동기 작업 수행
  // 성공 시: resume(Effect.succeed(결과))
  // 실패 시: resume(Effect.fail(에러))
});
```

아래와 같이 취소 상황에서는 cleanup effect를 return 합니다.

```tsx
const writeFileWithCleanup = (filename: string, data: string) =>
  Effect.async<void, Error>((resume) => {
    const writeStream = NodeFS.createWriteStream(filename);

    writeStream.write(data);
    writeStream.on("finish", () => resume(Effect.void));
    writeStream.on("error", (err) => resume(Effect.fail(err)));

    // 취소 시 정리(cleanup) 로직
    return Effect.sync(() => {
      console.log(`Cleaning up ${filename}`);
      NodeFS.unlinkSync(filename);
    });
  });
```

### Running Effects

Effect를 실행하기 위해서는 `run` 함수를 호출해야 합니다. \*Effect는 thunk로 작성되기 때문

Synchronously run : runSync, runSyncExit

runSync : sync Effect를 실행

```tsx
import { Effect } from "effect";

const program = Effect.sync(() => {
  console.log("Hello, World!");
  return 1;
});

const result = Effect.runSync(program);
// Output: Hello, World!

console.log(result);
// Output: 1
```

runSyncExit : Exit type(Exit.Success, Exit.Failure) Effect를 실행

runPromise : Effect를 Promise로 변환하여 작성할때

```tsx
import { Effect } from "effect";

Effect.runPromise(Effect.succeed(1)).then(console.log);
// Output: 1
```

runPromiseExit **:** Exit으로 resolve 되는 Promise를 반환하는 Effect를 실행

### **runFork** : The Default for Effect Execution

Effect를 실행하기 위한 가장 기본 함수입니다. observed 되거나 interrupted 될 수 있는 “fiber”를 반환합니다.

```tsx
import { Effect, Console, Schedule, Fiber } from "effect";

//      ┌─── Effect<number, never, never>
//      ▼
const program = Effect.repeat(
  Console.log("running..."),
  Schedule.spaced("200 millis")
);

//      ┌─── RuntimeFiber<number, never>
//      ▼
const fiber = Effect.runFork(program);

setTimeout(() => {
  Effect.runFork(Fiber.interrupt(fiber));
}, 500);
```

### **Best Practices**

Effect는 Synchronous vs. Asynchronous 여부를 타입 시스템에서 구분하지 않습니다. 타입 시스템을 단순핟게 유지하기 위한 의도적인 설계입니다.

Effect<Success, Error, Requirements> 는 Synchronous, Asynchronous 두 방식으로 사용 가능합니다.

Effect 실행시에 기본적으로 비동기 실행(runPromise, runFork)을 우선하며 동기 실행(runSync)는 특수한 경우에만 제한적으로 사용하는 것을 권장합니다.

### Refs

- https://effect.website/docs/getting-started/why-effect/
- https://effect.website/docs/getting-started/running-effects/

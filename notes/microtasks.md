### 태스크와 마이크로태스크

이 문서는 태스크와 마이크로태스크에 대하여 더 자세하게 다룬다.

### 태스크

[소개](./intro.md) 글에서 다루었듯이, 태스크는 작업/콜백 큐에 적재되고 런타임에 의해 콜백 큐에서 실행 스택으로 옮겨진다.  
타이머가 만료되면 콜백 큐에 실행될 함수(태스크)를 적재하는 `setTimeout`를 호출하는 경우를 예로 들 수 있다.

node.js를 사용한 [구체적인 예시](../lib/task.js)를 보자.

```console
$ env NODE_DEBUG=timer node lib/task.js 
main...
TIMER 3924762: no 1 list was found in insert, creating a new one
main...done
TIMER 3924762: process timer lists 145
TIMER 3924762: timeout callback 1
something
TIMER 3924762: 1 list empty
```
node.js에서 `setTimeout` 함수는 **libuv**의 `uv_timer_start`를 사용하여 구현되었다.

### 마이크로태스크

[소개](./intro.md) 글에서 마이크로태스크에 대한 약간의 배경 지식을 다루었다.  
이 글에서는 내부 구현을 살펴보자.

[여기](../test/microtask_test.cc)에서 마이크로태스크 예제를 확인할 수 있다.

Isolate 클래스는 큐잉 및 마이크로태스크 실행과 관련된 여러 기능을 제공한다.

```c++
class V8_EXPORT Isolate {                                                       
 public:
 ...
  V8_DEPRECATE_SOON("Use PerformMicrotaskCheckpoint.")                          
  void RunMicrotasks() { PerformMicrotaskCheckpoint(); }  
  void PerformMicrotaskCheckpoint();
  void EnqueueMicrotask(Local<Function> microtask);
  void EnqueueMicrotask(MicrotaskCallback callback, void* data = nullptr);
};
```

마이크로태스크 함수를 큐에 등록했을 때 어떤 일이 일어나는지 보자.

```console
$ lldb -- ./test/microtask_test
(lldb) br s -f microtask_test.cc -l 33
(lldb) r
```
`Isolate::EnqueueMicrotask` 는[`src/api/api.cc`](https://github.com/v8/v8/blob/main/src/api/api.cc)에서 확인할 수 있다.   
실제 호출은 결국 [`microtask-queue.cc`](../test/microtask_test.cc)에서 끝난다. ([`src/execution/microtask-queue.cc`](https://github.com/v8/v8/blob/main/src/execution/microtask-queue.cc)에서 확인 가능)

```c++
void MicrotaskQueue::EnqueueMicrotask(v8::Isolate* v8_isolate,
                                      v8::Local<Function> function) {
  Isolate* isolate = reinterpret_cast<Isolate*>(v8_isolate);
  HandleScope scope(isolate);
  Handle<CallableTask> microtask = isolate->factory()->NewCallableTask(
      Utils::OpenHandle(*function), isolate->native_context());
  EnqueueMicrotask(*microtask);
}
```
NewCallableTask는 [`src/heap/factory.cc`](https://github.com/v8/v8/blob/main/src/heap/factory.cc)에서 확인할 수 있다.

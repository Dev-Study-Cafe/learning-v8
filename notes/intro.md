## 소개

V8은 기본적으로 힙의 메모리 관리와 콜 스택(실행 스택)으로 구성된다. (매우 간단하지만 요점을 설명하는 데 도움이 된다).  
콜백 큐, 이벤트 루프 그리고 크롬에서는 WebAPIs (DOM, ajax, setTiimeout 등), 노드에서는 Node.js APIs도 있다.

```
+------------------------------------------------------------------------------------------+
| Google Chrome                                                                            |
|                                                                                          |
| +----------------------------------------+          +------------------------------+     |
| | Google V8                              |          |            WebAPIs           |     |
| | +-------------+ +---------------+      |          |                              |     |
| | |    Heap     | |     Stack     |      |          |                              |     |
| | |             | |               |      |          |                              |     |
| | |             | |               |      |          |                              |     |
| | |             | |               |      |          |                              |     |
| | |             | |               |      |          |                              |     |
| | |             | |               |      |          |                              |     |
| | +-------------+ +---------------+      |          |                              |     |
| |                                        |          |                              |     |
| +----------------------------------------+          +------------------------------+     |
|                                                                                          |
|                                                                                          |
| +---------------------+     +---------------------------------------+                    |
| |     Event loop      |     |          Task/Callback queue          |                    |
| |                     |     |                                       |                    |
| +---------------------+     +---------------------------------------+                    |
|                             +---------------------------------------+                    |
|                             |          Microtask queue              |                    |
|                             |                                       |                    |
|                             +---------------------------------------+                    |
|                                                                                          |
|                                                                                          |
+------------------------------------------------------------------------------------------+
```

콜 스택은 프레임 포인터의 스택이다.  
호출된 함수는 스택에 쌓인다.  
함수가 반환될 때, 스택에 저장된 해당 함수와 관련된 값들이 삭제된다.  
함수가 내부에서 다른 함수를 호출한다면, 그것 또한 스택에 쌓인다.  
내부 함수가 모두 반환되어야 해당 지점부터 실행이 계속 진행된다.  
함수 중 하나가 시간이 소요되는 작업을 수행한다면, 해당 함수가 반환되고 스택에서 삭제되기 전까지 이후 작업은 진행되지 않는다.  
이것이 싱글 스레드 프로그래밍 언어의 특징이다.

So that describes synchronous functions, what about asynchronous functions?  
Lets take for example that you call setTimeout, the setTimeout function will be
pushed onto the call stack and executed. This is where the callback queue comes
into play and the event loop. The setTimeout function can add functions to the
callback queue. This queue will be processed by the event loop when the call
stack is empty.

### Task

A task is a function that can be scheduled by placing the task on the callback
queue. This is done by WebAPIs like `setTimeout` and `setInterval`.
When the event loop starts executing tasks it will run all the tasks that
are currently in the task queue. Any new tasks that get scheduled by WebAPI
function calls are only pushed onto the queue but will not be executed until
the next iteration of the event loop.

When the execution stack is empty all the tasks in the microtask queue will be
run, and if any of these tasks add tasks to the microtask queue that will also
be run which is different compared with how the task queue handles this situation.

In Node.js `setTimeout` and `setInterval`...

### Micro task

Is a function that is executed after current function has run after all the
other functions that are currently on the call stack.

Microtasks internals info can be found in [microtasks](./microtasks.md).

#### Microtask queue

When a promise is created it will execute right away and if it has been resovled
you can call `then` on it.

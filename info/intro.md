# intro.md (2022/05/09)

execution stack: 실행 스택 (콜스택이라고도 부른다.)  
frame pointer: 함수가 실행되면, 콜 스택에 함수 컨텍스트가 생성된다. 이렇게 콜 스택에 할당되는 메모리 블록을 스택 프레임이라고 한다. 프레임 포인터는 이 스택 프레임의 주소값을 가리킨다.  
microtask queue: 마이크로태스크에 대하여 알게 되었다. 자바스크립트는 태스크와 마이크로태스크 중 마이크로태스크를 먼저 실행한다. **이벤트 루프는 마이크로태스트 큐의 모든 태스크를 처리한 후에 태스크 큐로 넘어간다.** 태스크의 대표적인 예는 setTimeout이고, 마이크로태스크의 대표적인 예는 Promise의 콜백이다. 자세한 내용은 [여기](https://baeharam.netlify.app/posts/javascript/JS-Task%EC%99%80-Microtask%EC%9D%98-%EB%8F%99%EC%9E%91%EB%B0%A9%EC%8B%9D)에서 확인할 수 있다.

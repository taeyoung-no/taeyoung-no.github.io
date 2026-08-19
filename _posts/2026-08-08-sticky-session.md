---
title: "로드 밸런서가 sticky session을 사용해야 했던 이유"
date: 2026-08-08 12:00:00 +0900
---

## 서론
[Chat App](https://github.com/taeyoung-no/chat-app) 프로젝트에서 서버를 두 대 띄우기 위해 로드 밸런서를 담당할 Nginx를 도입했습니다. `nginx.conf`를 제대로 작성했는지 검사하기 위해 디버깅이 쉬운 Least Connections를 사용해서 테스트해본 결과 Nginx가 정상적으로 로드 밸런싱하는 것을 확인했습니다. 그리고 서버가 여러 대여도 세션 방식의 인증과 서로 다른 서버에 붙은 유저끼리 실시간 통신을 하기 위해 Redis도 도입했습니다. 그런데 브라우저에서 테스트 결과 회원가입, 로그인, 방 생성 및 삭제 모두 정상 작동하는데 방에 입장하는 순간 `xhr post error`가 발생합니다. 이 에러를 해결하기 위해 겪은 시행착오를 기록했습니다.

---

## 본론
서버를 늘리고 로드 밸런서를 도입하자 방에 입장하는 순간, 즉 Socket.IO에 연결하는 순간 `xhr post error`가 발생했습니다. Nginx는 의도한 대로 작동하는 것을 확인했기 때문에 Socket.IO의 연결 방식에 대해 이해가 부족했던 것으로 판단, 공식 문서를 조사했습니다.

> By default, the client establishes the connection with the HTTP long-polling transport. [How it works](https://socket.io/docs/v4/how-it-works/)

처음에는 long-polling으로 연결합니다. WebSocket을 지원하지 않는 환경에서의 호환성 때문입니다. 서버가 여러 대인 경우 long-polling으로 통신하려면 사용자는 모든 요청을 같은 서버에 전송해야 합니다. 즉, 서버가 여러 대인데 로드 밸런서가 Least Connections 알고리즘을 사용하면 long-polling 통신에 실패할 수 있습니다. 

따라서 sticky session을 사용해야 할 것으로 판단, 공식 문서에 이 부분이 언급돼 있는지 찾아봤습니다. [Using multiple nodes](https://socket.io/docs/v4/using-multiple-nodes/#no-sticky-sessions-required)에 sticky session이 필요하며 `nginx.conf`에 `ip_hash`, 즉 sticky session을 사용하라고 명시돼 있습니다.

---

## 결론
Nginx가 sticky session을 사용하도록 수정했더니 `xhr post error`가 더 이상 발생하지 않았습니다.

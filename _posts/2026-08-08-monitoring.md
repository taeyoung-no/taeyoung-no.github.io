---
title: "인생 첫 모니터링: 메모리 누수 검사"
date: 2026-08-08 16:15:00 +0900
category: Chat App
---

## 서론
[Chat App](https://github.com/taeyoung-no/chat-app) 프로젝트에 학습 목적으로 모니터링을 도입해서 메모리 누수를 발견했습니다.

---

## 본론
이 프로젝트에서 방 목록 실시간 업데이트를 위해 Server-Sent Event(SSE)를 사용합니다. 클라이언트가 SSE 연결 시 응답 객체를 `Set`에 추가합니다. 

> The WeakSet is weak, meaning references to objects in a WeakSet are held weakly. If no other references to a value stored in the WeakSet exist, those values can be garbage collected. [WeakSet](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/WeakSet)

> A WeakRef object contains a weak reference to an object, which is called its target or referent. A weak reference to an object is a reference that does not prevent the object from being reclaimed by the garbage collector. In contrast, a normal (or strong) reference keeps an object in memory. [WeakRef](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/WeakRef)

`WeakSet`은 약한 참조라고 명시한 걸로 봐서 `Set`은 강한 참조를 사용한다고 유추했습니다. 따라서 클라이언트가 SSE 연결 해제 시 `Set`에서 응답 객체를 제거하지 않으면 메모리 누수가 발생할 것으로 판단했습니다. 실제로 메모리 누수가 발생하는지 확인하기 위해 프로젝트에 모니터링 도구를 도입했습니다.

### 모니터링 도구로 Prometheus와 Grafana를 선택한 이유
학습용 프로젝트이기 때문에 현업에서 자주 사용하는 도구를 도입합니다. Datadog 등 유명한 도구가 많지만 유료입니다. 오픈소스이며 인기도 많은 Prometheus로 메트릭을 수집, Grafana로 시각화해서 모니터링 경험을 쌓았습니다.

### 메모리 누수 확인
`Set`에 응답 객체를 추가할 때마다 메트릭에 크기를 기록, Prometheus는 해당 메트릭을 수집, Grafana는 수집된 메트릭을 시각화합니다. 메모리 누수가 없다면 새로고침 시 기존 응답 객체 제거 후 새 응답 객체를 추가합니다. 결과적으로 크기는 새로고침 전과 동일합니다. 메모리 누수가 있다면 응답 객체를 추가하기만 해서 크기가 증가합니다. 연결 해제 시 의도적으로 `Set`에서 제거하지 않도록 구현한 후 테스트했습니다. 결과적으로 `Set`의 크기가 새로고침 횟수에 비례하는 것을 확인했습니다. SSE 연결 해제 시 응답 객체를 제거하도록 변경했으며 같은 방법으로 모니터링을 통해 메모리 누수가 없어진 것을 확인했습니다.

---

## 결론
모니터링 도구를 학습하면서 `Set`이 강한 참조를 사용하는 것과 SSE 연결 해제 시 응답 객체를 직접 제거해야 한다는 것을 알게 됐습니다.

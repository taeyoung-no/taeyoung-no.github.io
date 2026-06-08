---
title:  "[Detached Element] no longer a child of this node"
date:   2026-06-04 20:14:00 +0900
---
## 서론
[Todo App](https://github.com/taeyoung-no/todo-app) 프로젝트 진행 중 inline editing을 구현하고 있었습니다. 수정 이모지(✏️)를 클릭하면 특정 요소가 `input` 요소로 교체됩니다. `input` 요소에서 `Enter` 입력 혹은 `blur` 발생 시 다시 원래 요소로 교체합니다.

```js
const save = () => {
      const span = document.createElement('span')
      span.className = 'todo-title'
      span.textContent = input.value.trim() || text
      input.replaceWith(span)
    }

    input.addEventListener('keydown', (e) => {
      if (e.key == 'Enter') save()
    })
    
    input.addEventListener('blur', save)
```

`input` 요소 바깥 클릭해서 `blur` 발생하면 정상 작동하지만 `Enter` 입력 시 에러가 발생했습니다. `replaceWith()`의 동작을 정확히 이해하지 못해서 겪은 시행착오를 다룹니다.

---
## 시행착오
정확한 에러 메시지는 `Uncaught NotFoundError: Failed to execute 'replaceWith' on 'Element': The node to be removed is no longer a child of this node.`입니다. 

### 이미 제거된 input에 대해 replaceWith()가 호출됐나?
`keydown`에 의해 `save()`가 호출돼서 `input`이 제거됐는데 알 수 없는 이유로 한 번 더 호출돼서 에러가 발생했다고 가정했습니다.
어떤 경로로 두 번째 `save()`가 호출되는지 알아내기 위해 코드를 점검했습니다. 그런데 만약 `input`이 제거된 후 `save()`가 호출됐다면 `input.value`가 `null`이기 때문에 `null` 참조 에러가 먼저 발생해야 한다는 것을 발견했습니다. 즉, `input`은 제거되지 않았습니다.

### replaceWith()의 정확한 동작은?
`input`은 제거되지 않았다는 것을 확인한 후 `replaceWith()`에 대해 조사했습니다.

> The `Element.replaceWith()` method replaces this `Element` in the children list of its parent with a set of `Node` objects or strings. Strings are inserted as equivalent `Text` nodes. [MDN Web Docs](https://developer.mozilla.org/en-US/docs/Web/API/Element/replaceWith)

위와 에러 메시지를 고려해서 다음처럼 가설을 세웠습니다. `Element.replaceWith(param1)`는 `Element` 자체를 제거하는 게 아니라 고아로 만드는 것입니다. 즉, 고아에 대해 `replaceWith()`를 호출해서 에러가 발생하는 것입니다. 검증하기 위해 `input.value`와 `input.parentNode`를 출력해봤는데 전자는 정상적으로 출력됐으며 후자는 `null`이었습니다. 즉, 가설이 옳았습니다.

### 해결하기
어쨌든 `replaceWith()`가 두 번 호출돼서 에러가 발생합니다. `replaceWith()`에 의해 `input`이 고아가 되면서 `blur` 발생, 결과적으로 두 번째 `replaceWith()`가 발생하는 것으로 판단했습니다. `keydown` 입력 시 `blur` 이벤트 리스너를 제거하도록 수정해서 해결했습니다. 

---
## 결론
`replaceWith()`는 요소 자체를 제거하지 않고 고아로 만드는 것입니다. 조사해보니 detached element라고 부릅니다.
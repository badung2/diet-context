# Diet Context — 한국어 설명

`diet-context`는 긴 LLM/agent 작업에서 불필요하게 커지는 active context(활성 컨텍스트)를 줄이기 위한 범용 Skill이다.

핵심은 두 가지다.

1. **Tool-result offloading(도구 결과 외부화)**  
   긴 로그, 테스트 결과, 명령 출력 같은 대용량 중간 결과를 context에 계속 들고 다니지 않고 파일/아티팩트로 보존한 뒤, active context에는 요약 + 경로 + 필요한 retrieval hint(재조회 힌트)만 남긴다.

2. **Structured handoff / compaction(구조화 인계/압축)**  
   길어진 대화나 workflow history(작업 이력)를 통째로 다음 단계에 넘기는 대신 현재 목표, 완료된 작업, 제약, 중요한 결정, 다음 단계, 관련 파일만 구조화해서 새 working baseline(작업 기준점)으로 만든다.

목적은 단순히 토큰을 무조건 줄이는 것이 아니다.

> 앞으로 여러 번 반복해서 들고 갈 필요가 없는 정보는 active context에서 빼되, 나중에 필요하면 복구할 수 있게 만든다.

---

## 왜 필요한가

긴 agent workflow는 보통 다음처럼 context가 계속 누적된다.

```text
system/workflow instruction
+ 이전 대화
+ 이전 tool call
+ shell output
+ build/test log
+ research notes
+ 이전 agent 결과
+ 현재 작업
```

문제는 과거 정보가 완전히 쓸모없지 않더라도, 다음 단계에서 그 전체가 동시에 필요하지는 않은 경우가 많다는 점이다.

Diet Context는 다음 질문을 한다.

```text
지금 반드시 active context에 있어야 하는 것은 무엇인가?
요약해도 되는 것은 무엇인가?
원문을 밖에 보존하고 필요할 때 일부만 다시 읽어도 되는 것은 무엇인가?
```

---

## Tool-result offloading

예를 들어 테스트 로그가 50 KB라면 다음 agent에게 50 KB 전체를 그대로 넘기지 않는다.

대신:

```text
Artifact: .diet-context/test-run.txt
Summary: 182 tests, 179 passed, 3 failed
Key details:
- auth.test 실패
- router.test fallback assertion 실패
- cache.test stale cache 실패
Retrieval hints:
- "FAILED" 검색
- auth.test 오류 주변만 읽기
```

처럼 넘긴다.

중요한 점은 **파일로 뺐다가 다음 agent가 파일 전체를 다시 읽으면 절감 효과가 크게 줄어든다**는 것이다.

따라서 가능한 경우:

```text
grep / search
특정 line range
특정 section
head / tail
오류 부분만 추출
```

같은 selective retrieval(선택적 재조회)을 우선한다.

---

## Structured handoff / compaction

긴 작업 이력을 다음처럼 정리한다.

```text
Goal
Constraints
Progress
  Done
  In Progress
  Blocked
Key Decisions
Next Steps
Critical Context
Artifacts / Relevant Files
```

예를 들어 60k짜리 과거 context를 계속 들고 가는 대신, 다음 작업에 필요한 상태만 5k 정도의 handoff로 만들고 거기서 새로 시작하는 식이다.

이 과정에는 handoff를 생성하고 읽는 비용이 추가되지만, 여러 후속 턴이 60k 과거 context를 반복해서 들고 가는 비용보다 작을 때 이득이 난다.

---

## dynamic-workflow와의 관계

이 Skill은 orchestration(오케스트레이션)을 담당하지 않는다.

역할 분리는 다음과 같다.

```text
dynamic-workflow
→ 누구에게, 언제, 어떤 순서로 일을 위임할지 결정

diet-context
→ 그 과정에서 어떤 context를 얼마나 줄여서 넘길지 결정
```

따라서 `diet-context`는 스스로 `delegate_task`를 호출하기 위해 존재하지 않는다.

다른 workflow가 이미 delegation(위임)을 결정했을 때 전달되는 payload(전달 컨텍스트)를 최적화하는 역할이다.

---

## 고정 임계값을 쓰지 않는 이유

`8 KB 이상이면 무조건 offload` 같은 규칙은 단순하지만 정확하지 않다.

30 KB짜리 반복 로그는 쉽게 줄여도 되지만, 10 KB짜리 정밀한 법률 문서나 수치 결과는 원문 유지가 더 중요할 수 있다.

따라서 Diet Context는 다음을 보고 모델이 판단하도록 설계한다.

- 앞으로 몇 번 더 재사용될 가능성이 있는가
- 정보 밀도가 높은가 낮은가
- 후속 단계가 얼마나 남았는가
- 원문을 다시 찾을 수 있는가
- 정확한 wording/ordering/value가 중요한가
- 요약으로 정보 손실이 발생할 위험이 큰가
- 요약과 재조회 비용이 실제 절감보다 더 큰가

애매하면 더 많이 보존한다.

---

## Recoverable compression(복구 가능한 압축)

목표는 삭제가 아니라 **복구 가능한 압축**이다.

가능하면 다음을 유지한다.

- 정확한 파일 경로
- 정확한 identifier
- 중요한 command
- 중요한 error text
- 중요한 숫자
- 미해결 질문
- provenance(어떤 tool/file/command가 정보를 만들었는지)
- 원문을 다시 찾을 수 있는 metadata

즉:

```text
"provider에 문제가 있었다"
```

보다는:

```text
"Provider X가 effort=high probe 중 30초 후 HTTP 429 반환"
```

처럼 continuation(작업 지속)에 필요한 정밀도를 유지한다.

---

## 언제 안 쓰는가

다음과 같은 경우에는 굳이 공격적으로 줄이지 않는다.

- 곧 끝날 짧은 작업
- exact wording이 중요한 작업
- 수치/정렬/포맷을 그대로 보존해야 하는 작업
- artifact를 안정적으로 저장하거나 다시 읽을 수 없는 환경
- 요약 후 결국 원문 대부분을 다시 읽어야 하는 상황
- 요약 비용이 절감 효과보다 큰 상황

작은 작업에서는 아무것도 하지 않는 것이 가장 효율적일 수 있다.

---

## 설치 개념

repo root 자체가 Skill이다.

```text
diet-context/
├── README.md
├── README.ko.md
├── SKILL.md
└── agents/
   └── openai.yaml
```

사용하는 agent runtime의 global Skill directory에 repository root를 clone/copy/link하고 `SKILL.md`가 discoverable(탐색 가능)한지 확인하면 된다.

환경마다 Skill 경로가 다르므로 이 프로젝트가 특정 절대 경로를 강제하지는 않는다.

Hermes에는 GitHub URL과 함께 다음처럼 요청하면 된다.

```text
이 repository의 README와 SKILL.md를 먼저 읽어 역할과 동작 경계를 파악해.
그 다음 이 환경의 전역 Skill로 설치해.
dynamic-workflow는 수정하지 말고 diet-context와 함께 동작하도록 유지해.
설치 후 Skill이 discoverable한지 검증해.
```

---

## 출처 / Inspiration

이 Skill의 설계는 open-source [Mixdog](https://github.com/tribgames/mixdog)에서 확인한 context-efficiency(컨텍스트 효율화) 아이디어, 특히 큰 tool output의 외부화, context compaction, selective retrieval 패턴에서 일부 영감을 받았다.

다만 `diet-context`는 Mixdog runtime의 포트가 아니며 Mixdog 코드나 Mixdog 실행환경을 필요로 하지 않는 독립적인 policy Skill이다.
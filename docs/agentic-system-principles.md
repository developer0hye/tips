# 효과적인 Agentic System을 만드는 5가지 원칙

Agentic system은 LLM이 **tool을 호출하고 → 결과를 보고 → 다음 행동을 스스로 결정하는 loop**다. 데모는 하루면 만든다. 문제는 그 다음이다. 요청이 늘고, tool이 늘고, 모델이 바뀌고, 어느 날 agent가 같은 tool을 400번 호출하고 있는 걸 청구서로 알게 된다.

전통적인 소프트웨어와 다른 점은 세 가지다.

| 차이 | 의미 |
|---|---|
| **Non-deterministic** | 같은 입력에 다른 경로를 간다. "재현해서 고친다"가 잘 안 된다 |
| **비용이 실행 경로에 비례** | 버그가 곧 돈이다. 무한 loop는 CPU가 아니라 청구서를 태운다 |
| **핵심 부품이 몇 달마다 교체된다** | 오늘 최적화한 prompt가 다음 모델에서 무의미해진다 |

이 세 성질을 감당하려면 다음 다섯 원칙을 지켜야 한다.

| 원칙 | 답하는 질문 | 안 지키면 |
|---|---|---|
| **Scalability** | 요청과 작업 복잡도가 10배 되면 버티나? | context 폭발, 비용 폭발, latency 폭발 |
| **Modularity** | 한 부품만 갈아끼울 수 있나? | prompt 한 줄 고치면 전체가 흔들린다 |
| **Continuous Learning** | 운영에서 나온 실패가 개선으로 돌아오나? | 같은 실패를 매주 반복한다 |
| **Resilience** | tool이 죽고 모델이 헛소리해도 살아남나? | 한 번의 timeout이 작업 전체를 날린다 |
| **Future-proofing** | 모델이 바뀌어도 시스템이 남나? | 특정 모델의 버그 우회책이 부채로 쌓인다 |

---

## 1. Scalability — Context는 유한한 자원이다

Agent의 확장은 두 축이다. **Load**(동시 요청 수)와 **complexity**(한 작업의 step 수, context 크기). Load는 일반 서버와 같은 방법으로 풀린다. 진짜 문제는 complexity다.

Context window는 메모리와 같다. 차오르면 느려지고, 비싸지고, **품질이 떨어진다.** 긴 context에서 모델은 중간에 있는 정보를 놓치고(lost in the middle), 초반에 한 지시를 잊는다. 토큰 하나가 곧 latency이고 비용이고 정확도다.

### 실천

**작업을 쪼개고 context를 격리한다.** Orchestrator가 큰 작업을 나누고, 각 sub-agent는 자기 몫만 보고 결과만 돌려준다. 파일 30개를 읽어야 하는 조사 작업을 sub-agent에 맡기면, main context에는 30개 파일이 아니라 결론 한 문단만 남는다.

**상태를 밖으로 뺀다.** 대화 history를 프로세스 메모리에 쌓지 말고 외부 저장소에 둔다. Worker가 stateless가 되면 horizontal scaling이 그냥 된다.

**상한을 건다.** 요청마다 max steps, max tokens, max cost를 정한다. 상한이 없는 loop는 언젠가 반드시 돈다.

**모델을 tiering한다.** 분류, routing, 요약 같은 쉬운 단계는 작은 모델로, 판단이 필요한 단계만 큰 모델로. 비용의 대부분은 "큰 모델이 사소한 일을 하는" 데서 나온다.

**Caching한다.** System prompt, tool 정의, 참조 문서처럼 매 turn 반복되는 prefix는 prompt caching 대상이다. Tool 호출 결과도 같은 인자면 다시 부르지 않는다.

**비동기로 처리한다.** 몇 분 걸리는 작업을 HTTP 요청 하나에 묶어두지 않는다. Queue에 넣고, 진행 상태를 저장하고, 완료를 알린다.

### Anti-pattern

- Tool 40개를 한 agent에 다 넣기. Tool 정의만으로 context 수천 토큰을 먼저 쓰고, 모델은 비슷한 이름의 tool을 헷갈리기 시작한다
- Tool 결과를 그대로 context에 붓기. 10MB 로그 파일을 읽으라는 tool 호출 하나가 세션을 끝낸다. Tool이 잘라서 돌려줘야 한다
- "일단 다 넣고 모델이 알아서 하겠지." Context가 길수록 모델은 덜 알아서 한다

---

## 2. Modularity — 한 부품만 갈아끼울 수 있어야 한다

Agent를 하나의 prompt 덩어리로 보면 안 된다. 최소 여섯 부품으로 나뉜다.

| 부품 | 역할 | 교체 주기 |
|---|---|---|
| **Model** | 추론 | 몇 달 |
| **Prompt** | 역할, 규칙, 형식 | 매주 |
| **Tools** | 외부 세계와의 접점 | 기능 추가마다 |
| **Memory** | 세션 안/밖 상태 | 드묾 |
| **Orchestration** | loop 제어, 분기, 병렬 | 드묾 |
| **Evaluation** | 잘 됐는지 판정 | 실패 사례마다 |

교체 주기가 전부 다르다. 이걸 한 파일에 섞어두면 매주 바뀌는 prompt를 고칠 때마다 드물게 바뀌는 orchestration이 깨진다.

### 실천

**Tool은 single responsibility, 좁은 interface.** `do_everything(action, payload)` 같은 tool은 모델도 사람도 못 쓴다. Tool 이름과 설명만 보고 무슨 일이 일어날지 알 수 있어야 한다. 입력과 출력은 schema로 고정한다.

**Tool interface는 표준을 따른다.** MCP처럼 모델과 tool 사이의 protocol을 표준화하면, tool을 다른 agent에서 재사용하고 모델을 바꿔도 tool은 그대로 쓴다.

**Prompt를 코드처럼 다룬다.** 파일로 분리하고, 버전을 매기고, 리뷰를 거친다. Prompt가 코드 문자열 안에 박혀 있으면 아무도 diff를 안 본다. ([Versioning의 중요성](versioning.md) 참고)

**역할별로 agent를 나눈다.** 조사하는 agent, 구현하는 agent, 검증하는 agent는 system prompt도 tool 목록도 다르다. 하나로 합치면 각각의 지시가 서로를 희석한다.

**Business logic은 코드에 둔다.** "금액이 100만 원 넘으면 승인 필요"는 prompt에 쓰는 게 아니라 코드로 강제한다. Prompt는 확률적으로 지켜지고, 코드는 항상 지켜진다.

### Anti-pattern

- Tool 안에서 또 LLM을 호출하고 그 안에서 또 tool을 부르는 숨은 재귀. 비용과 latency가 어디서 나오는지 아무도 모르게 된다
- System prompt에 예외 케이스를 하나씩 덧붙여서 300줄이 된 것. 각 줄이 왜 있는지 아무도 모르고, 지우면 뭐가 깨질지 몰라서 못 지운다
- 모델 ID를 코드 곳곳에 hardcoding. 모델 하나 바꾸는 데 grep부터 해야 한다

---

## 3. Continuous Learning — 측정 없이는 학습도 없다

여기서 "학습"은 fine-tuning만을 말하는 게 아니다. **운영에서 나온 신호가 시스템 개선으로 되돌아오는 feedback loop**가 있느냐의 문제다. 대부분의 팀에는 이 loop가 없다. Production 실패를 Slack에서 논의하고, prompt를 한 줄 고치고, 그게 다른 케이스를 깨뜨렸는지는 아무도 모른다.

학습은 세 층으로 일어나고, 아래층이 위층의 전제다.

```
3층  Model 개선     fine-tuning, distillation      ← 마지막 수단, 가장 비쌈
2층  System 개선    prompt / tools / orchestration ← 실패 사례가 eval로 들어와야 가능
1층  Runtime memory 세션 간 사실, 사용자 feedback   ← 즉시 효과, 오염 위험
────────────────────────────────────────────────────
0층  Observability  모든 호출의 입력/출력/비용/latency ← 이게 없으면 위 전부 불가능
```

### 실천

**0층: 전부 기록한다.** 모든 모델 호출과 tool 호출의 입력, 출력, 토큰, latency, 비용을 trace로 남긴다. 어느 요청이 왜 비쌌는지, 어느 step에서 틀어졌는지 이게 없으면 추측밖에 못 한다. Non-deterministic 시스템에서 trace는 로그가 아니라 **유일한 재현 수단**이다.

**2층: 실패를 eval로 바꾼다.** Production에서 틀린 케이스는 그날 eval dataset에 들어가야 한다. 그러면 다음 prompt 수정이 그 케이스를 고쳤는지, 다른 케이스를 깨뜨렸는지 숫자로 나온다. Regression test와 같은 개념이다.

**Online metric을 정한다.** 성공률, human intervention rate, 평균 step 수, retry rate, 요청당 비용. 이 숫자가 dashboard에 없으면 개선했는지 악화됐는지 감으로 판단하게 된다.

**1층: Memory는 검증하고 만료시킨다.** Agent가 세션 간에 기억을 쌓는 건 강력하지만, **잘못된 것을 배우는 것도 학습이다.** Memory에 출처와 시점을 붙이고, 오래된 것은 만료시키고, 틀린 것은 지울 수 있어야 한다. 한 번 잘못 저장된 "사용자는 X를 선호한다"가 이후 모든 세션을 오염시킨다.

**3층은 마지막에.** Prompt와 tool로 안 풀리는 문제가 데이터로 확인됐을 때만 fine-tuning을 고려한다. 그 전에 하면 무엇을 고쳤는지 모르는 채 모델 하나를 더 관리하게 된다.

### Anti-pattern

- Trace 없이 "가끔 이상하게 답해요"를 디버깅하려는 것. 재현이 안 되니 고칠 수도 없다
- Eval 없이 prompt 수정. 고쳤다는 느낌만 있고 증거는 없다
- Memory에 무엇이든 저장. 3개월 뒤 agent가 왜 그렇게 행동하는지 아무도 설명 못 한다

---

## 4. Resilience — 실패는 예외가 아니라 정상이다

Agent loop에서는 실패가 여러 곳에서 동시에 온다.

| 실패 | 증상 | 방어 |
|---|---|---|
| 모델 timeout, rate limit | 요청 전체가 멈춤 | retry + exponential backoff, fallback model |
| Tool 오류 (네트워크, 권한) | stack trace로 죽음 | **오류를 모델에 돌려준다** — 모델이 우회할 기회를 준다 |
| Hallucinated tool 인자 | 없는 파일, 잘못된 ID로 호출 | schema validation, 존재 확인 후 실행 |
| 무한 loop | 같은 tool을 같은 인자로 반복 | step 상한, 반복 감지, 비용 상한 |
| 긴 작업 중 crash | 처음부터 다시 | checkpoint, 재개 가능한 상태 저장 |
| 위험한 행동 | 파일 삭제, 결제, 외부 발송 | approval gate, sandbox |
| 부분 성공 | 절반 하고 멈춤 | graceful degradation: 된 만큼 돌려주고 사람에게 넘김 |

### 실천

**Tool 오류는 모델에게 보여준다.** Tool이 실패하면 예외를 던져 프로세스를 죽이는 게 아니라, 오류 메시지를 tool result로 돌려준다. 모델은 "파일이 없다"는 메시지를 보면 다른 경로를 찾는다. 그 기회를 코드가 빼앗으면 안 된다. 단, 같은 오류가 세 번 반복되면 멈추게 한다.

**Retry는 idempotent한 tool에만.** 결제 API를 timeout 났다고 retry하면 두 번 결제된다. Tool마다 idempotency를 표시하고, 아닌 tool은 retry 대신 사람에게 묻는다.

**상한은 여러 겹으로.** Step 수, 토큰, wall-clock time, 비용. 하나만 있으면 그걸 피해가는 실패 모드가 나온다.

**위험한 행동은 분류하고 gate를 둔다.** 읽기는 자유, 쓰기는 로그, 삭제와 외부 발송은 승인. Agent에게 주는 권한은 최소로 시작해서 필요할 때 넓힌다(least privilege). 역방향은 사고가 난 뒤에야 일어난다.

**격리한다.** 코드를 실행하는 agent는 sandbox 안에서 돈다. Prompt injection으로 agent가 이상한 명령을 실행해도 피해 범위가 sandbox 안에서 끝나야 한다.

**긴 작업은 재개 가능하게.** 20 step 작업의 15 step에서 죽었으면 15 step부터 다시 시작해야 한다. 각 step의 결과를 외부에 저장하고, 재시작 시 읽어 들인다.

### Anti-pattern

- 모든 tool에 무조건 3회 retry. Idempotent하지 않은 tool에서 사고가 난다
- 오류를 삼키고 빈 결과 반환. 모델은 "성공했는데 결과가 없다"고 해석하고 잘못된 결론을 낸다
- Agent에 처음부터 admin 권한. "나중에 줄이자"는 오지 않는다

---

## 5. Future-proofing — 모델은 바뀌고, 데이터는 남는다

모델은 몇 달마다 세대가 바뀐다. 이 사실이 설계에 주는 함의는 하나다. **모델에 맞춘 것은 전부 소모품이고, 모델과 무관한 것만 자산이다.**

| 소모품 (모델 바뀌면 다시) | 자산 (모델 바뀌어도 남음) |
|---|---|
| 특정 모델용 prompt trick | Eval dataset |
| 모델 약점을 우회하는 코드 | Trace와 실패 사례 |
| 모델별 parameter tuning | Tool과 그 interface |
| | Orchestration 구조 |
| | 도메인 지식이 담긴 문서 |

### 실천

**모델은 설정값이다.** 모델 ID, parameter, endpoint를 한 곳에서 관리한다. 코드는 "모델"이라는 추상 interface만 본다. 바꿀 때 코드 수정이 필요하면 이미 잘못됐다.

**모델 교체에 gate를 둔다.** 새 모델로 바꾸기 전에 eval set을 전부 돌린다. 점수가 떨어진 케이스를 보고 결정한다. 이게 가능한 팀과 불가능한 팀의 차이는 3번 원칙을 지켰느냐다.

**모델의 약점을 코드로 우회할 때는 표시를 남긴다.** "이 모델은 JSON을 잘 못 뱉으니 후처리로 고친다"는 코드는 다음 모델에서 불필요하거나 해가 된다. 왜 있는지, 어느 모델 때문인지 주석으로 남기고, 모델 교체 때 지울 후보로 올린다. 이걸 안 하면 아무도 못 지우는 workaround가 쌓인다.

**모델에 더 맡기는 방향으로 설계한다.** 모델이 못 해서 코드로 hardcoding한 규칙은 모델이 좋아질수록 발목을 잡는다(the bitter lesson). 지금 모델이 70%밖에 못 하는 일이라면 30%를 코드로 막는 대신, 검증과 retry로 감싸는 게 다음 모델에서 더 잘 살아남는다.

**표준 protocol을 쓴다.** Tool 연결, 메시지 형식, trace 포맷에서 특정 vendor 전용 방식보다 표준을 택한다. Vendor를 바꿀 일이 없어도, 표준을 따르면 생태계의 tool을 그대로 가져다 쓸 수 있다.

**Auditable하게 만든다.** Agent가 무엇을 왜 했는지 나중에 추적할 수 있어야 한다. 규제와 보안 요구는 시간이 갈수록 늘기만 한다. 지금 trace를 남기지 않으면 나중에 소급해서 만들 수 없다.

### Anti-pattern

- 특정 모델의 출력 형식에 regex로 의존. 모델이 바뀌면 parser가 깨지고, 왜 깨졌는지 찾는 데 하루 걸린다
- "이 모델은 이렇게 해야 잘 된다"는 팁이 prompt 곳곳에 박혀 있는데 출처 없음
- Eval 없이 모델 업그레이드. "더 좋은 모델이니까 더 잘 되겠지"는 자주 틀린다

---

## 다섯 원칙은 서로 당긴다

전부 동시에 최대로 만족시킬 수는 없다.

| Trade-off | 내용 | 판단 기준 |
|---|---|---|
| Modularity ↔ Scalability | Agent를 잘게 나누면 hop이 늘고 latency와 비용이 는다 | Context 격리 효과가 hop 비용보다 큰가 |
| Resilience ↔ 비용 | Retry, fallback, validation은 전부 토큰이다 | 실패 비용이 방어 비용보다 큰가 |
| Continuous Learning ↔ 안정성 | Memory와 자동 개선은 예측 불가능성을 키운다 | 되돌릴 수 있는가 |
| Future-proofing ↔ 지금 성능 | 모델 특화 trick은 지금 점수를 올린다 | 표시하고 만료 계획이 있는가 |

우선순위를 하나만 정하라면 **observability(tracing)가 먼저다.** 다섯 원칙 중 어느 것도 측정 없이는 지켜졌는지 확인할 수 없다. Scalability는 비용 그래프로, resilience는 실패율로, learning은 eval 점수로, future-proofing은 모델 교체 전후 비교로 확인된다. 전부 trace에서 나온다.

## Checklist

**Scalability**
- [ ] 요청당 max steps / tokens / cost 상한이 있다
- [ ] 큰 조사 작업은 sub-agent로 격리되어 main context에 결론만 남는다
- [ ] Tool 결과는 tool이 잘라서 돌려준다
- [ ] 반복되는 prompt prefix는 caching된다

**Modularity**
- [ ] Prompt가 코드와 분리된 파일이고 버전 관리된다
- [ ] Tool마다 schema가 있고 이름만 봐도 하는 일을 안다
- [ ] 반드시 지켜야 할 규칙은 prompt가 아니라 코드에 있다
- [ ] 모델 ID가 한 곳에만 있다

**Continuous Learning**
- [ ] 모든 모델/tool 호출이 trace로 남는다
- [ ] Production 실패 사례가 eval set에 들어가는 절차가 있다
- [ ] Prompt 변경 전후 eval 점수를 비교한다
- [ ] Memory에 출처와 시점이 있고 지울 수 있다

**Resilience**
- [ ] Tool 오류가 모델에 결과로 전달된다
- [ ] Retry는 idempotent한 tool에만 적용된다
- [ ] 삭제, 결제, 외부 발송에 approval gate가 있다
- [ ] 코드 실행은 sandbox 안에서 일어난다
- [ ] 긴 작업은 중간부터 재개할 수 있다

**Future-proofing**
- [ ] 모델 교체 전 eval을 전부 돌린다
- [ ] 모델 약점 workaround 코드에 이유와 대상 모델이 적혀 있다
- [ ] Tool 연결은 표준 protocol을 쓴다
- [ ] Agent의 모든 행동을 사후에 추적할 수 있다

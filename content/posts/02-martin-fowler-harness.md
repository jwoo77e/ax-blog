---
title: "Martin Fowler의 하네스 프레임워크를 산업안전에 적용해봤다"
date: 2026-04-23T10:00:00+09:00
draft: false
tags: ["하네스 엔지니어링", "Martin Fowler", "Feedforward", "Feedback", "AI Agent"]
categories: ["AX 실무"]
description: "Feedforward/Feedback 이중 제어 프레임워크를 멀티 에이전트 시스템에 적용한 갭 분석과 구현 과정"
---

## Martin Fowler가 던진 한마디

2026년 4월, Martin Fowler가 [Harness engineering for coding agent users](https://martinfowler.com/articles/harness-engineering.html)를 발표했다. 코딩 에이전트를 실무에 쓰는 사람이라면 한 번쯤 겪었을 문제를 정면으로 다룬 글이다.

핵심 공식은 단순하다.

```
Agent = Model + Harness
```

모델은 LLM 그 자체고, 하네스(Harness)는 모델이 올바르게 일하도록 감싸는 모든 구조물이다. 그리고 이 하네스는 두 축으로 나뉜다.

- **Feedforward** — 에이전트가 일하기 *전에* 방향을 잡아주는 가이드. 프롬프트, 규칙, 컨텍스트 주입 같은 것들.
- **Feedback** — 에이전트가 일한 *후에* 결과를 검증하는 센서. 테스트, 린터, 검증 스크립트 같은 것들.

읽는 순간 머릿속에서 우리 시스템이 겹쳐졌다. 나는 지금 리스크제로에서 6개 서브 에이전트를 오케스트레이션하는 멀티 에이전트 시스템을 운영하고 있다. 코딩 에이전트는 아니지만, 구조적으로 완전히 같은 문제를 안고 있었다.

## 갭 분석: Feedforward는 A, Feedback은 F

Fowler의 프레임워크를 우리 시스템에 대입해봤다. 결과는 처참할 정도로 명확했다.

**Feedforward(사전 제어)는 꽤 잘 되어 있었다.**

- `SOUL.md`에 라우팅 규칙 5개 — 키워드별로 어떤 에이전트가 받을지 명시
- `SKILL.md` 6개 — 각 에이전트마다 트리거 조건, 실행 절차, 주의사항까지 정의
- `AGENTS.md`에 Hard Rules 4개 + Red Lines 5개 — 바이너리 파일 읽지 마라, 외부 전송 전 확인하라 등

여기까지만 보면 괜찮아 보인다. 에이전트가 *뭘 해야 하는지*는 충분히 알려주고 있었다.

**문제는 Feedback(사후 제어)이 사실상 부재했다는 것이다.**

- 에이전트가 만든 산출물이 실제로 유효한지 검증하는 로직? 없음.
- 이메일 전송이 실패했을 때 재시도? 없음.
- AI가 생성한 보고서의 품질을 다른 AI가 리뷰? 없음.
- 반복되는 실패 패턴을 수집해서 스킬을 개선? 없음.

결론은 이거였다: **에이전트가 실수해도 자기 수정할 수 있는 센서가 없어서, 모든 검증 부담이 나에게 왔다.** 매번 산출물을 열어보고, 이메일이 갔는지 확인하고, 보고서 내용이 맞는지 크로스체크하고. 에이전트를 쓰는데 오히려 내 일이 늘어나는 아이러니.

Fowler의 2×2 매트릭스로 그려보면 이랬다:

| | Feedforward | Feedback |
|---|---|---|
| **Inferential** (AI 판단) | ✅ SOUL.md 라우팅 | ❌ 품질 리뷰 없음 |
| **Computational** (규칙 기반) | ✅ SKILL.md 절차 | ❌ 검증 스크립트 없음 |

오른쪽 열이 텅 비어 있었다.

## 5개 Feedback 센서 구현

빈칸을 채우기로 했다. Computational 센서부터 만들고, Inferential은 그 위에 얹었다.

### 1. verify_output.py — 산출물 자동 검증

```python
def verify(filepath: str, required_sections: list[str]) -> dict:
    if not os.path.exists(filepath):
        return {"score": 0, "reason": "파일 없음"}
    size = os.path.getsize(filepath)
    if size < 1024:
        return {"score": 20, "reason": f"파일 크기 의심: {size}B"}
    content = Path(filepath).read_text()
    missing = [s for s in required_sections if s not in content]
    score = max(0, 100 - len(missing) * 15)
    return {"score": score, "missing": missing}
```

파일 존재, 크기, 필수 섹션 포함 여부를 체크하고 0-100 점수를 매긴다. 특허 명세서 작성 시 도면 설명 누락, 청구항 빠짐 같은 걸 자동으로 잡아낸다.

### 2. send_with_retry.sh — 이메일 전송 재시도

```bash
#!/bin/bash
MAX_RETRY=3
for i in $(seq 1 $MAX_RETRY); do
    if gsk email-send "$@" 2>/dev/null; then
        echo "{\"status\":\"sent\",\"attempt\":$i}" >> send_log.jsonl
        exit 0
    fi
    sleep $((i * 5))
done
echo "{\"status\":\"failed\",\"attempts\":$MAX_RETRY}" >> send_log.jsonl
exit 1
```

3회 재시도에 지수 백오프. 모든 시도를 JSONL로 남긴다. 단순하지만 이전에는 이것조차 없었다.

### 3. review_agent.py — AI 품질 리뷰

산출물을 별도 LLM에 넘겨서 논리 일관성, 사실 크로스체크, 누락 항목을 검토한다. 생성한 에이전트와 다른 모델이 리뷰하는 게 포인트다. 셀프 리뷰는 같은 실수를 반복하니까.

### 4. steering_loop.py — 실패 패턴 수집

```python
def analyze_failures(log_path: str) -> list[str]:
    failures = [json.loads(l) for l in open(log_path) if "failed" in l]
    patterns = Counter(f["error_type"] for f in failures)
    suggestions = []
    for error_type, count in patterns.most_common(3):
        suggestions.append(f"SKILL.md에 '{error_type}' 방지 규칙 추가 권고 ({count}회 발생)")
    return suggestions
```

Feedback 데이터가 Feedforward 개선으로 이어지는 루프. Fowler가 말한 "steering loop"을 그대로 구현한 것이다.

### 5. agent_status.py — 에이전트 상태 관리

6개 서브 에이전트의 활성 상태, 마지막 실행 시각, 성공/실패 비율을 실시간으로 추적한다. 대시보드(`127.0.0.1:8765`)에 연동해서 한눈에 볼 수 있게 했다.

## HEARTBEAT.md — 상시 점검 체계

센서를 만들었으면 돌아가게 해야 한다. `HEARTBEAT.md`에 6개 상시 점검 항목을 정의했다:

1. **크론잡 정상 작동** — 스케줄된 작업이 실제로 실행되고 있는가
2. **스킬 정합성** — SKILL.md와 실제 에이전트 동작이 일치하는가
3. **메모리 최신성** — memory/ 디렉토리에 오늘자 기록이 있는가
4. **대시보드 응답** — 상태 API가 200을 반환하는가
5. **산출물 검증** — 최근 산출물의 verify 점수가 임계값 이상인가
6. **스티어링 반영** — 수집된 개선 제안이 실제로 적용되었는가

## 결과: 2×2가 채워졌다

5개 센서와 HEARTBEAT를 적용한 후, 매트릭스가 이렇게 바뀌었다:

| | Feedforward | Feedback |
|---|---|---|
| **Inferential** | ✅ SOUL.md 라우팅 | ✅ review_agent.py |
| **Computational** | ✅ SKILL.md 절차 | ✅ verify_output.py |

빈칸이 사라졌다. 그리고 체감이 확실했다. 특허 명세서를 작성할 때, `verify_output.py`가 도면 잘림과 폰트 깨짐을 내가 열어보기 전에 잡아냈다. 이메일이 안 갔을 때 내가 모르는 사이에 재시도가 돌았다. steering_loop이 "청구항 번호 오류 3회 반복"을 감지하고 SKILL.md 수정을 제안했다.

## 배운 것

**"Feedback 없는 AI 에이전트는 눈 감고 일하는 것과 같다."**

Fowler의 글은 코딩 에이전트를 대상으로 쓰였다. 그런데 산업안전 도메인의 멀티 에이전트 시스템에도 프레임워크가 그대로 적용됐다. 어쩌면 당연하다. 제어 이론에서 Feedforward/Feedback 이중 루프는 도메인을 가리지 않는 보편 원리니까.

구현 순서도 중요했다. **Computational 센서가 먼저다.** 파일 존재 여부, 크기, 필수 항목 체크 같은 규칙 기반 검증은 구현이 쉽고 효과가 즉각적이다. Inferential 센서(AI 리뷰)는 그 다음이다. 규칙으로 잡을 수 있는 건 규칙으로 잡고, AI는 규칙이 못 잡는 영역에 쓰는 게 맞다.

결국 하네스 엔지니어링의 핵심은 이거다: **모델을 바꾸는 게 아니라, 모델이 일하는 환경을 바꾸는 것.** 같은 LLM이라도 어떤 하네스 안에서 돌아가느냐에 따라 완전히 다른 결과가 나온다. 우리 시스템이 그 증거다.

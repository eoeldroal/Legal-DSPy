# DSPy + EXAONE-4 Legal QA System: v1→v2 개발 기록

## 개요

DSPy 프레임워크와 EXAONE-4 모델을 결합한 법률 질의응답 시스템 개발 기록. V1(50% 정답률)에서 V2(86% 정답률)로의 성능 개선 과정과 해결된 기술적 문제들을 기록.

## V1 버전: 초기 구현

### 기본 구조
```python
lm = dspy.LM('openai/exaone-4-32b', api_key='no', api_base='http://0.0.0.0:8005/v1')
COT = dspy.ChainOfThought('passage, question, options, statements -> answer : int')
```

### 발견된 문제들

**문제 1: EXAONE-4 Thinking 능력 미활용**
- EXAONE-4의 `enable_thinking=True` 기능이 비활성화됨
- 모델의 내재적 `<think>...</think>` 추론 능력 사용 불가

**문제 2: 두 추론 시스템 간 충돌**
- DSPy 형식: `[[ ## reasoning ## ]]`, `[[ ## answer ## ]]`
- EXAONE 형식: `<think>...</think>`
- 결과: 중복되고 혼재된 출력 형식

```
<think>
[[ ## reasoning ## ]] 사용자가 요청한...
[[ ## answer ## ]] 답은 2번입니다.
[[ ## completed ## ]]
</think>

[[ ## reasoning ## ]] 다시 한번 분석하면...
[[ ## answer ## ]] 2
```

**문제 3: 파싱 시스템 오작동**
- `[[ ## completed ## ]]` 마커가 `<think>` 블록 내부에 출현
- DSPy 파서가 thinking 내용을 최종 답변으로 오인식

**문제 4: 성능 제한**
- 형식 충돌로 인한 모델 혼란
- 정답률 50% 수준에 정체

## V2 버전: 문제 해결

### 해결 과정 1: EXAONE Thinking 활성화

DSPy.LM의 `extra_body` 파라미터를 통한 thinking 모드 활성화:

```python
lm = dspy.LM(
    'openai/exaone-4-32b', 
    api_key='no', 
    api_base='http://0.0.0.0:8005/v1',
    extra_body={"chat_template_kwargs": {"enable_thinking": True}},
    temperature=0.6,
    top_p=0.95
)
```

### 해결 과정 2: 시그니처 재설계

**핵심 변경: reasoning 필드 제거**

```python
class ExaoneLegal(dspy.Signature):
    """Leet 문제를 정확히 맞춰야 한다."""
    passage = dspy.InputField(desc="법률 관련 지문")
    question = dspy.InputField(desc="분석할 질문") 
    options = dspy.InputField(desc="선택지")
    statements = dspy.InputField(desc="관련 보기")
    
    final_answer : int = dspy.OutputField(
        desc="1~5 사이의 정수로 구성된 최종 답안 번호. 맨 마지막에 한 번만 제시하면 된다.",
        format=int
    )
```

**설계 원칙:**
- EXAONE-4가 `<think>` 블록에서 자유롭게 추론
- DSPy는 최종 답안만 구조화
- 두 시스템 간 역할 분리

### 결과: 이상적인 출력 형식

```
<think>
First, I need to answer the question: "윗글의 내용과 일치하지 않는 것은?" 
[자유로운 추론 과정...]
So, option ② is the one that doesn't match.
Therefore, the answer should be ②.
Now, for the output, I need to provide final_answer with the integer.
</think>

[[ ## final_answer ## ]]
2
```

**성과:**
- thinking 블록: 형식 제약 없는 자유로운 추론
- 출력 블록: DSPy 형식 준수
- 모델의 자율적 모드 전환 인식

### 성능 개선

- V1: 50% 정답률
- V2: 86% 정답률
- 개선: +36 포인트 절대 향상

## 핵심 교훈

1. **시스템 고유성 존중**: 각 AI 시스템의 특성을 억압하지 않고 활용
2. **역할 분리**: EXAONE(thinking) + DSPy(구조화)의 명확한 분담
3. **최소 제약**: 불필요한 형식 요구사항 제거로 성능 향상
4. **인지적 일관성**: 자유로운 사고 → 구조화된 출력의 자연스러운 흐름

## 미래 발전 방향

### 1단계: Tool 통합
**Code 도구 결합**
- 복잡한 법률 계산을 위한 Python 코드 생성/실행
- `<think>` 블록에서의 자유로운 법리 해석 유지

**Thinking 도구 결합**
- 다층적 추론 시스템 구축
- 1차 thinking → 도구 기반 검증 → 최종 종합 판단

### 2단계: 고도화된 프롬프트 최적화
**복합 시스템 최적화**
- EXAONE thinking + 도구 사용 + DSPy 구조의 삼중 조화
- V2 핵심 원칙을 복잡한 시스템에서도 유지
- MIPROv2를 활용한 전체 시스템 최적화

**목표**
- 법률 AI의 문제 해결 도구 → 법률 분석 파트너로 진화
- 직관적 판단력과 체계적 분석 능력의 결합
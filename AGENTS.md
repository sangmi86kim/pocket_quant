# PocketQuant CLI Backend Harness v0.1

> 이 문서는 **현재 코드 기준**으로 작성된 사양서다. (코드가 source of truth)
> "향후 확장 로드맵" 항목만 아직 미구현이며, 그 외는 실제 코드와 일치한다.

## 목표

PocketQuant의 첫 MVP를 위한 CLI-only 백엔드 하네스.

전략 포켓몬을 생성하고, 여러 체육관에서 생존 테스트를 수행한 뒤 결과를 출력한다.

## MVP 범위

- GUI 없음
- DB 없음
- 외부 API 없음
- 실제 금융 데이터 없음 (전투는 랜덤 보정값으로 대체 — 적합도 함수 교체점)
- LLM 없음
- 김박사/오박사 없음
- CLI 실행만 지원

---

## 프로젝트 구조

```text
pocket_quant/
├─ main.py                # CLI 진입점 + 결과 출력
├─ app/
│  ├─ __init__.py
│  └─ backend/
│     ├─ __init__.py
│     ├─ models.py        # 데이터 모델 + 점수/등급 테이블
│     ├─ strategy.py      # 전략 생성 + 이름 자동 생성
│     ├─ gym.py           # 체육관 정의
│     ├─ battle.py        # 전투 계산 (생존/사망 판정)
│     └─ evolve.py        # 단일목적 GA (개체군/선택/교배/돌연변이/세대)
├─ README.md
├─ README.html
└─ AGENTS.md
```

> 참고: 출력 포맷은 별도 `report.py` 없이 `main.py` 안에서 처리한다.
> `tests/`는 아직 없음 (향후 추가 예정).

---

## models.py

데이터의 '모양'만 정의한다. 로직은 최소.

### 점수/등급 테이블 (모듈 상수)

```python
GENE_SCORES = {"DD": 20, "RSI": 15, "MA": 25, "BB": 10, "FX": 5}
ALL_GENES   = list(GENE_SCORES.keys())   # ["DD", "RSI", "MA", "BB", "FX"]
GRADES      = [(0.9, "S"), (0.7, "A"), (0.5, "B"), (0.3, "C"), (0.0, "D")]
```

> 유전자는 Enum이 아니라 **문자열 + dict 점수표**로 관리한다 (MVP 단순화).

### Strategy dataclass

```python
genes: list[str]      # 예) ["DD", "RSI"]
name: str = ""        # 자동 생성된 이름
# base_score() -> int : 유전자 점수 합 (필드가 아니라 메서드)
```

### Gym dataclass

```python
name: str
difficulty: int
volatility: int       # 현재는 연출용 정보, 판정에는 미사용
```

### BattleResult dataclass

```python
gym_name: str
score: int            # 최종 점수 (base_score + 랜덤 보정)
survived: bool
```

### Report dataclass

```python
strategy: Strategy
results: list[BattleResult]
# @property: survive_count, death_count, survive_rate, grade
```

---

## strategy.py

### 역할

전략 생성 및 이름 자동 생성.

### 함수

```python
create_strategy(gene_count: int | None = None) -> Strategy
make_name(genes: list[str]) -> str
```

### 규칙

* gene_count가 없으면 1~5개 랜덤
* 유전자 중복 없음 (`random.sample`)
* 유전자 점수 합산 (`Strategy.base_score()`)
* 이름 자동 생성

### 이름 규칙

```text
RSI몬
DD-RSI몬
DD-RSI-MA몬
```

* 약 20% 확률로 특수 이름 부여 (예: "타이탄 드래곤") — **이미 구현됨**
  * TITLES = ["ATH", "디아블로", "헤르메스", "타이탄"]
  * SUFFIXES = ["몬", "드래곤", "킹", "마스터"]

---

## gym.py

### 기본 체육관 (4개)

값은 실제 역사 데이터 근거. difficulty(생존 난이도 = 낙폭×기간×회피불가)와
volatility(VIX 피크)는 **서로 비례하지 않는다**.

```text
FINANCIAL_CRISIS    difficulty = 90   volatility = 80   # S&P -57%, 시스템 붕괴, VIX~80 → 최난도
DOTCOM              difficulty = 85   volatility = 55   # -49%(나스닥-78%), 2.5년 느린 약세장, VIX~45
RATE_SHOCK          difficulty = 60   volatility = 40   # -25%, 채권 동반하락이나 질서있음, VIX~36
COVID               difficulty = 40   volatility = 95   # -34%지만 V자 즉시회복(생존쉬움), VIX 82.7 역대최고
```

> boss / TRUMP 체육관은 v0.1에 없음 (향후 확장 시 고려).

### 함수

```python
all_gyms() -> list[Gym]
```

---

## battle.py

### 유전자 점수

```text
DD    +20
RSI   +15
MA    +25
BB    +10
FX    +5
```

> 점수표의 실제 출처는 `models.GENE_SCORES`. battle은 이를 참조만 한다.

### 전투 로직

```text
random_bonus = random(-20 ~ +20)        # RANDOM_SWING = 20
final_score  = strategy.base_score() + random_bonus

final_score >= gym.difficulty  → 생존
final_score <  gym.difficulty  → 사망
```

> 이 `random_bonus`가 **"랜덤 계측값 주입" 지점**이다.
> 나중에 yfinance 실데이터 백테스트로 교체하면 = "실계측 돌리기".

### 함수

```python
fight(strategy: Strategy, gym: Gym) -> BattleResult
challenge(strategy: Strategy, gyms: list[Gym]) -> Report
```

---

## evolve.py (단일목적 GA)

### 역할

전략 개체군을 여러 세대 진화시켜 "전 체육관 평균 생존률"이 가장 높은 전략을 찾는다.

### 적합도 (단일목적)

```text
fitness = 전 체육관 평균 생존률 (0~1, 숫자 하나)
        = mean( 체육관별 생존률 )
평가는 trials회 반복 도전 후 평균 (랜덤 보정 노이즈 제거)
```

### GA 4단계 + 함수

```python
evaluate(strategy, gyms, trials) -> {"fitness", "per_gym"}   # 1. 평가
select(scored, keep) -> list[Strategy]                       # 2. 선택(절단)
crossover(genes_a, genes_b) -> list[str]                     # 3. 교배(균등)
mutate(genes, rate) -> list[str]                             # 4. 돌연변이(추가/제거)
evolve(gyms, pop_size, generations, trials, on_generation) -> (best, stats)
```

* `on_generation(gen, best, stats)` : 세대별 콜백 훅 (로깅/시각화/향후 early stop)

### 한계 (의도된 것)

* 현재 점수 체계에선 '유전자 많을수록 유리' → **전 유전자 조합으로 수렴**.
* 목적은 (1) GA 기계 검증, (2) 최적 전략조차 어느 시장에서 박살나는지 관찰.
* 이 관찰이 다목적(NSGA-III)·타입상성 필요성의 근거가 된다.

---

## main.py

### 역할

CLI 진입점 + 결과 출력 (별도 report 모듈 없음).

### argparse 옵션

```bash
# [단판 모드]
-g, --genes        # 전략 유전자 개수 지정 (생략 시 랜덤)

# [진화 모드 — 단일목적 GA]
--evolve           # 진화 모드 실행
--pop              # 개체군 크기 (기본 20)
--generations      # 세대 수 (기본 10)
--trials           # 전략당 평가 도전 횟수 = 평균낼 표본 (기본 20)

# [공통]
--seed             # 랜덤 시드 고정 (재현 가능 → GA 검증용)
```

### 실행 예시

```bash
python main.py                                              # 단판
python main.py -g 3
python main.py --evolve                                     # 진화(GA)
python main.py --evolve --pop 20 --generations 8 --seed 42  # 재현 가능한 진화
```

> 반드시 프로젝트 루트에서 실행. (`app.backend....` 절대 경로 import 구조)

### 출력 예시 (실제 출력)

```text
=== PocketQuant ===

전략 생성
DD + RSI + MA
이름: DD-RSI-MA몬

체육관 도전

DOTCOM
사망

COVID
생존

RATE_SHOCK
사망

결과
생존 1
사망 2
등급 C
```

### 등급 규칙

```text
S: 생존률 >= 90%
A: 생존률 >= 70%
B: 생존률 >= 50%
C: 생존률 >= 30%
D: 그 외
```

---

## 개발 원칙

### 유지

* 단순한 CLI MVP
* 실행 가능한 코드 우선
* 200~300줄 내외
* 구조는 확장 가능하게

### 금지

* GUI
* DB
* 웹서버
* 실제 주가 데이터
* 투자 API
* LLM
* 김박사/오박사
* 과도한 추상화

---

## 구현 완료

### v0.2 단일목적 GA  ✅
* 개체군 + 선택(절단) + 교배(균등) + 돌연변이 + 세대 반복 (`evolve.py`)
* `--seed`로 재현 가능, 세대별 콜백 훅, 시장별 박살 로그
* 교배/돌연변이/세대(원래 v0.3~v0.5 항목)를 단일목적 MVP로 통합 구현

---

## 향후 확장 로드맵 (미구현)

### 다음: 다목적 최적화 (NSGA-III)

* 단일 적합도(평균) → **체육관별 생존력 = 각각의 목적**
* 결과 = 최강 전략 1개가 아니라 **전략 Pareto front** (국면별 특화 + 올라운더)
* **단, 순서 주의**: 단일목적 MVP로 "시장별 박살 패턴"을 먼저 관찰한 뒤,
  그 충돌을 보고 목적함수를 정의한다. (책상에서 미리 짜지 않음)

### 정비

* `tests/` 추가 (test_strategy / test_gym / test_battle / test_evolve)
  * 전략 생성·중복 없음·gene_count 일치 / 체육관 목록·difficulty 존재
  * fight 결과 타입·survived bool·seed 재현성 / GA 수렴·교배·돌연변이

### 전략 도감

* 전략 저장 / 전적 기록 / 생존률 누적

### v1.0 김박사 NPC

* 룰베이스 대사
* 이후 LLM 연결

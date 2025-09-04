지금까지 준비하신 *방법2* 구조를 존중하면서 **연결(배치↔DB↔시트↔백엔드 API↔프런트)**가 틈 없이 맞물리도록 보강했습니다.

---

## 0) 한눈에 보기 — 무엇을 만들 것인가

- **프롬프트 연구소(ABn 실험 파이프라인)**
    - 카테고리별 프롬프트 **1~10버전 동시 실험** → 요약 생성 → 지표 계산 → DB 저장 → Google Sheets 로그/대시보드 자동 반영.
    - **읽ㅎ기 API**와 **관리 API**를 통해 프런트의 “연구소” 화면에 **버전별 성능 집계/리더보드** 제공, 관리자만 프롬프트 CRUD.
    - **매일 2회** 스케줄, 실패 시 폴백/재시도.
    - (선택) ADHOC 실행 단추(버튼)로 즉시 실행.

---

## 1) 산출물(Deliverables) & 파일 목록

### A. 파이썬 배치 레포 (요약·평가·시트 반영)

> 이미 제안된 구조를 기본으로 하되 운영 편의용 몇 가지만 추가합니다.
> 

```
repo/
 ├─ docker/
 │   └─ Dockerfile
 ├─ requirements.txt
 ├─ .env.example
 ├─ sql/
 │   ├─ 00_schema.sql
 │   ├─ 10_indexes.sql
 │   └─ 30_views.sql                  # [신규] 조회용 뷰
 ├─ app/
 │   ├─ config.py
 │   ├─ db.py
 │   ├─ metrics.py
 │   ├─ summarizer_client.py
 │   ├─ sheets.py
 │   ├─ pipeline.py
 │   ├─ server.py                     # [선택] ADHOC 실행 HTTP 엔드포인트
 │   └─ __init__.py
 ├─ scripts/
 │   ├─ seed_prompts.py               # [신규] 프롬프트 대량 시드
 │   ├─ sync_prompts_from_sheet.py    # [선택] 시트→DB 동기화
 │   └─ dryrun.sh                     # [신규] DRY-RUN 스크립트
 ├─ tests/
 │   ├─ test_metrics.py               # [신규] 지표 단위테스트(골든셋)
 │   └─ test_db_roundtrip.py          # [신규] upsert/idempotency 테스트
 └─ .github/workflows/batch.yml

```

**추가 SQL(조회 뷰)** — `sql/30_views.sql`

```sql
CREATE OR REPLACE VIEW v_lab_version_scores AS
SELECT e.prompt_id, p.category_name, p.version_no,
       AVG(e.final_score) AS avg_final, COUNT(*) AS n
FROM news_summary_eval e
JOIN prompt p ON p.prompt_id = e.prompt_id
GROUP BY e.prompt_id, p.category_name, p.version_no;

CREATE OR REPLACE VIEW v_lab_best_versions AS
SELECT category_name, version_no, avg_final, n,
       RANK() OVER (PARTITION BY category_name ORDER BY avg_final DESC) AS rnk
FROM v_lab_version_scores;

```

**ADHOC 실행 엔드포인트(선택)** — `app/server.py` (Flask 예시)

```python
from flask import Flask, request, jsonify
from app.pipeline import run_one_category
app = Flask(__name__)

@app.post("/run")
def run():
    category = request.args.get("category")
    if not category: return jsonify({"error":"category required"}), 400
    try:
        run_one_category(category)
        return jsonify({"ok": True, "category": category})
    except Exception as e:
        return jsonify({"ok": False, "error": str(e)}), 500

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8080)

```

> 주의: requirements.txt에 Flask==3.0.3를 추가해야 합니다(선택기능).
> 

---

### B. Spring Boot(news-service) — 연구소 API(읽기/관리)

**패키지 제안:** `com.yourorg.news.lab`

**도메인/리포지토리**

```
domain/
 ├─ Prompt.java
 ├─ ExpRun.java
 ├─ NewsSummaryV2.java
 └─ NewsSummaryEval.java
repository/
 ├─ PromptRepository.java
 ├─ ExpRunRepository.java
 ├─ NewsSummaryV2Repository.java
 └─ NewsSummaryEvalRepository.java

```

**서비스**

```
service/
 ├─ PromptService.java         # 프롬프트 CRUD, 중복/버전 규칙 검증
 ├─ LabQueryService.java       # v_lab_* 뷰 및 집계 쿼리
 └─ RunService.java            # 최근 run 조회(옵션: ADHOC 트리거 연계)

```

**컨트롤러**

```
controller/
 ├─ PromptAdminController.java  # /api/lab/prompts/**
 ├─ LabReadController.java      # /api/lab/evals/**, /runs/**
 └─ RunWebhookController.java   # [선택] /api/lab/runs/{id}/status

```

**DB 마이그레이션(Flyway 권장)**

```
resources/db/migration/
 ├─ V20250901__lab_schema.sql   # 00_schema.sql 그대로
 ├─ V20250901_1__lab_indexes.sql
 └─ V20250901_2__lab_views.sql  # 30_views.sql

```

**간단한 API 스펙(OpenAPI 조각)**

```yaml
# GET /api/lab/evals/overview?category=ECONOMY
# 응답: 카테×버전 평균 성능
- avg_final: number
  n: integer
  num_f1: number
  title_overlap: number
  category: string
  version_no: integer

# GET /api/lab/evals/leaderboard?category=ECONOMY&limit=10
# 응답: 상위 버전 리스트(평균점수, 표본수, 최신시각)
- prompt_id: number
  category: string
  version_no: number
  avg_final: number
  n: number
  last_run_at: string

# GET /api/lab/prompts?category=ECONOMY
- prompt_id, category_name, version_no, prompt_text, target_lines, status, created_at

# POST /api/lab/prompts
# body:
{ category_name, version_no, prompt_text, target_lines }
# 201 Created

# PATCH /api/lab/prompts/{id}/status
# body: { status: "ACTIVE" | "INACTIVE" }
# 200 OK

```

**시큐리티**

- `ROLE_ADMIN`: `/api/lab/prompts/**` (쓰기)
- `ROLE_USER` 이상: `/api/lab/evals/**`, `/api/lab/runs/**` (읽기)
- 게이트웨이 라우팅: `/api/lab/**` → news-service

---

### C. 프런트(Next.js) — 연구소 화면

```
app/admin/prompt-lab/page.jsx          # 대시보드 페이지
components/lab/PromptTable.jsx         # 프롬프트 버전 1~10 표
components/lab/EvalLeaderboard.jsx     # 카테고리별 상위 버전
components/lab/EvalTrendChart.jsx      # [선택] 버전별 추세(최근 runs)
components/lab/RunNowButton.jsx        # [선택] ADHOC 실행 버튼
lib/labService.js                      # 연구소 API 래퍼
.env.local                             # NEXT_PUBLIC_API_URL=...

```

**lib/labService.js(예시)**

```jsx
export async function getEvalOverview(category) {
  const r = await fetch(`${process.env.NEXT_PUBLIC_API_URL}/api/lab/evals/overview?category=${category}`, { credentials: "include" });
  if (!r.ok) throw new Error("overview failed");
  return r.json();
}
export async function getLeaderboard(category, limit=10) {
  const r = await fetch(`${process.env.NEXT_PUBLIC_API_URL}/api/lab/evals/leaderboard?category=${category}&limit=${limit}`, { credentials: "include" });
  if (!r.ok) throw new Error("leaderboard failed");
  return r.json();
}
export async function getPrompts(category) {
  const r = await fetch(`${process.env.NEXT_PUBLIC_API_URL}/api/lab/prompts?category=${category}`, { credentials: "include" });
  if (!r.ok) throw new Error("prompts failed");
  return r.json();
}

```

**page.jsx(요약)**

- 카테고리 셀렉트 → `getEvalOverview`, `getLeaderboard`, `getPrompts` 호출
- 표/차트 렌더링, 관리자 권한이면 프롬프트 토글/수정 버튼 노출

> 프런트의 기존 인증 fetch 유틸을 그대로 재사용해 Authorization 헤더가 자동 세팅되도록 하세요.
> 

---

### D. Google Sheets — 탭 구조(최소)

- `eval_log` *(파이프라인 자동 append)*
    - 컬럼: `ts, category, version_no, prompt_id, news_id, title, summary, final_score, num_f1, title_overlap, tfidf_cos, rouge1, rougeL, repeat_rate, halluc_ratio`
- `dashboard` *(파이프라인 자동 overwrite)*
    - 컬럼: `category, version_no, avg_final, n, num_f1, title`
- (선택) `prompts`
    - 컬럼: `category, version_no, prompt_text, target_lines, status`
    - `scripts/sync_prompts_from_sheet.py`로 DB upsert 가능(관리 편의)

---

## 2) 운영 플로우(연결이 제대로 되는지)

1. **배치 시작**(GitHub Actions Cron: 01:00 / 13:00 UTC ≒ 서울 10:00 / 22:00)
2. **DB 읽기**: `prompt`(활성 1..10), `news`(카테고리별 최신 N)
3. **요약 생성**: 요약 API 호출 실패 시 **로컬 더미 폴백**
4. **지표 계산**: `metrics.py`
5. **DB 적재**: `news_summary_v2`, `news_summary_eval` upsert (idempotent)
6. **시트 반영**: `eval_log` append, `dashboard` 갱신
7. **프런트/백엔드**:
    - 백엔드의 `LabReadController`가 뷰/집계를 **읽기 전용 API**로 노출
    - 프런트 `prompt-lab` 페이지가 API를 호출하여 **표·리더보드·(선택)추세** 표시
8. (선택) **ADHOC 실행**: 프런트 버튼 → 파이썬 `server.py`의 `/run?category=...` 호출

---

## 3) 품질·안전 설계(핵심 체크)

- **폴백/재시도**
    - 요약 API 30초 타임아웃, 예외 시 로컬 더미요약
    - 시트 업데이트 실패해도 배치 자체는 성공 처리(로그만)
- **성능/인덱스**
    - 뉴스 최신순 + 카테 필터 인덱스
    - 평가결과 조회용 뷰 `v_lab_*`
- **지표 신뢰도**
    - 숫자/날짜/금액 **정규식 F1** + **반복/할루/커버리지/ROUGE/TF-IDF**
    - (옵션) 임베딩 코사인 `ENABLE_EMBEDDINGS=true`일 때만
- **재현성**
    - `.env`의 카테고리·lines·기사 샘플 수 **고정**
    - (선택) `LAB_SEED` 추가 시 샘플링 결정론 보장
- **권한**
    - 프롬프트 쓰기(생성/수정/활성화 토글)는 **ADMIN 한정**
- **관찰성**
    - run_id, category, version_no 단위로 로그/지표가 **모두 연결**
    - (선택) Slack webhook: 상위 버전 변화 알림

---

## 4) 에이전트 작업 목록(태스크 브레이크다운)

1. **DB & 마이그레이션**
- [ ]  `00_schema.sql`, `10_indexes.sql`, `30_views.sql` 적용(Flyway로 이관)
- [ ]  시드: `scripts/seed_prompts.py` 또는 수동 insert
1. **배치 레포**
- [ ]  `pipeline.py` 지표/시트 반영 확인
- [ ]  (선택) `server.py` + `Flask` 의존성 추가
- [ ]  `tests/test_metrics.py`에 골든 케이스 5개 이상
1. **백엔드 API**
- [ ]  도메인/리포지토리/서비스/컨트롤러 생성
- [ ]  OpenAPI 명세 반영, Swagger UI 노출 `/swagger-ui.html`
- [ ]  시큐리티 룰/게이트웨이 라우팅 추가
1. **프런트**
- [ ]  `app/admin/prompt-lab/page.jsx` 초기 화면
- [ ]  `lib/labService.js` API 래퍼
- [ ]  관리자 권한 분기/버전 토글 UI
1. **CI/CD & 스케줄**
- [ ]  GH Actions `batch.yml` 시크릿 주입
- [ ]  (사설 DB면) self-hosted runner 또는 VPC 내 CronJob
1. **Google Sheets**
- [ ]  서비스계정 **편집자** 공유
- [ ]  `GOOGLE_SA_JSON` 경로 확인(런너/컨테이너에서 유효)

---

## 5) 수락 기준(Definition of Done)

- [ ]  매일 2회 자동 실행 후, **`news_summary_v2`/`news_summary_eval` 레코드 증가** 확인
- [ ]  같은 기사·프롬프트에 대해 **중복 실행 시 upsert**만 일어나며, 총계 변형 없음
- [ ]  `eval_log`에 당일 실행 로그 append, `dashboard`에 **카테×버전 평균** 갱신
- [ ]  **프런트 연구소 페이지**에서:
    - 카테고리 선택 → 버전별 평균 성능(표본수 포함) 표시
    - 리더보드 상위 10 버전 표시
    - (관리자) 프롬프트 활성/비활 토글 및 신규 등록 가능
- [ ]  요약 API 장애 상황에서도 **배치 실패 없이 폴백 동작**
- [ ]  OpenAPI 문서에서 명시한 API 5종이 **권한 정책대로** 동작

---

## 6) 실행·운영 메모

- 로컬 드라이런:
    
    ```bash
    cp .env.example .env && vi .env   # DB/Sheets 설정
    pip install -r requirements.txt
    bash scripts/dryrun.sh            # 카테 1종, 기사 3건만 시험
    python -m app.pipeline            # 전체 실행
    
    ```
    
- 프로덕션:
    - GH Actions의 `ENV_FILE`에 .env 원문 저장, `GOOGLE_SA_JSON` 원문 저장
    - 서울 기준 **10:00 / 22:00** 스케줄이 맞는지 UTC 크론 확인(`0 1,13 * * *`)

---

## 7) (선택) 추가 개선안

- **가중치 조정 UI**: `final_score` 가중치를 시트 `config` 탭에서 관리, 배치가 읽어 반영
- **통계검정**: 버전 간 평균 차이 Bootstrap/t-test(표본수 조건) 결과를 `dashboard`에 함께 기재
- **Counterfactual 셋**: 동일 기사 셋(7일 간 대표 100건)으로만 버전 비교 → 노이즈 감소
- **K8s CronJob**로 전환 시 VPC 내부 DB 접근·보안 간편화
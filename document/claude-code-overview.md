# ras-commander에서 Claude Code 활용 가이드

## ras-commander에서 Claude Code로 할 수 있는 것들

### 1. HEC-RAS 모델 실행 및 관리

#### 시작하기 전 필수 준비사항

HEC-RAS를 Claude Code로 실행하려면 다음 3가지가 필요합니다.

| 항목 | 설명 |
|------|------|
| **HEC-RAS 설치** | Windows에 HEC-RAS 설치 (보통 `C:/Program Files/HEC/HEC-RAS/<버전>/Ras.exe`) |
| **ras-commander 설치** | `pip install ras-commander` |
| **HEC-RAS 프로젝트** | `.prj` 파일이 있는 HEC-RAS 프로젝트 폴더 |

> **중요**: HEC-RAS는 Windows 전용입니다. macOS/Linux에서는 실행되지 않습니다.

---

#### Step 1: 패키지 설치 확인

```python
# ras-commander 설치
pip install ras-commander

# 설치 확인
import ras_commander
print(ras_commander.__version__)
```

---

#### Step 2: 예제 프로젝트로 시작하기 (권장)

실제 HEC-RAS 프로젝트가 없을 때 내장 예제 프로젝트를 사용합니다.

```python
from ras_commander import RasExamples

# 사용 가능한 예제 목록 확인
RasExamples.list_examples()

# 예제 프로젝트 추출 (예: Muncie)
project_path = RasExamples.extract_project("Muncie")
print(f"프로젝트 경로: {project_path}")
```

주요 예제 프로젝트:

| 이름 | 유형 | 설명 |
|------|------|------|
| `Muncie` | 1D/2D | 가장 기본적인 예제 |
| `BaldEagleCrkMulti2D` | 2D | 2D 메시 예제 |
| `Dam Breaching` | 비정상류 | Dam Breach 분석 |

---

#### Step 3: 프로젝트 초기화

```python
from ras_commander import init_ras_project, ras

# 방법 1: 추출한 예제 프로젝트 사용
project_path = RasExamples.extract_project("Muncie")
init_ras_project(project_path, "6.5")  # 두 번째 인수: 설치된 HEC-RAS 버전

# 방법 2: 기존 프로젝트 경로 직접 지정
init_ras_project("C:/Projects/MyModel", "6.6")

# 초기화 후 프로젝트 정보 확인
print(ras.plan_df)   # 플랜 목록
print(ras.geom_df)   # 지오메트리 목록
```

`init_ras_project()` 호출 후 전역 `ras` 객체에서 모든 DataFrame에 접근할 수 있습니다.

---

#### Step 4: 플랜 실행

```python
from ras_commander import RasCmdr

# 단일 플랜 실행 (가장 기본)
RasCmdr.compute_plan("01")

# 결과를 별도 폴더에 저장하면서 실행 (원본 보존)
RasCmdr.compute_plan("01", dest_folder="C:/output/run1")

# 여러 플랜 동시 병렬 실행
RasCmdr.compute_parallel(["01", "02", "03"], max_workers=2)

# 순차 실행 (디버깅용)
RasCmdr.compute_test_mode(["01", "02", "03"])
```

> **주의**: 플랜 번호는 반드시 문자열로 전달해야 합니다. `"01"` (O) / `1` (X)

---

#### Step 5: 실행 완료 확인 및 결과 추출

```python
from ras_commander.hdf import HdfResultsPlan

hdf_file = "C:/Projects/MyModel/MyModel.p01.hdf"

# 정상류/비정상류 자동 감지 후 결과 추출
if HdfResultsPlan.is_steady_plan(hdf_file):
    wse = HdfResultsPlan.get_steady_wse(hdf_file)
else:
    wse = HdfResultsPlan.get_wse(hdf_file, time_index=-1)  # 마지막 시간 스텝

print(f"수위 범위: {wse.min():.2f} ~ {wse.max():.2f} ft")
```

---

#### 전체 워크플로 요약 (복사해서 바로 사용 가능)

```python
from ras_commander import RasExamples, init_ras_project, RasCmdr
from ras_commander.hdf import HdfResultsPlan

# 1. 예제 프로젝트 추출
project_path = RasExamples.extract_project("Muncie")

# 2. 프로젝트 초기화
init_ras_project(project_path, "6.5")

# 3. 플랜 실행
RasCmdr.compute_plan("01")

# 4. 결과 추출
hdf_file = list(project_path.glob("*.p01.hdf"))[0]
wse = HdfResultsPlan.get_wse(hdf_file, time_index=-1)
print(f"최대 수위: {wse.max():.2f} ft")
```

---

#### 실행 모드 비교

| 모드 | 함수 | 사용 시점 |
|------|------|----------|
| **단일 실행** | `compute_plan("01")` | 플랜 1개, 일반 작업 |
| **병렬 실행** | `compute_parallel(["01","02"])` | 여러 플랜, 빠른 처리 |
| **순차 테스트** | `compute_test_mode(["01","02"])` | 디버깅, 리소스 제한 환경 |
| **원격 분산** | `compute_parallel_remote(...)` | 다수 머신에 분산 |

#### 주요 파라미터

| 파라미터 | 기본값 | 설명 |
|----------|--------|------|
| `dest_folder` | `None` | 결과 저장 폴더 (None이면 원본에 저장) |
| `num_cores` | `None` | CPU 코어 수 (None이면 HEC-RAS 자동 결정) |
| `force_rerun` | `False` | 결과가 최신이어도 강제 재실행 |
| `clear_geompre` | `False` | `.c##` 파일 삭제 후 재컴파일 |
| `force_geompre` | `False` | `.g##.hdf` + `.c##` 모두 삭제 후 완전 재처리 |
| `stream_callback` | `None` | 실시간 모니터링 콜백 |

> **스마트 스킵**: `force_rerun=False`(기본값)이면 입력 파일보다 HDF가 최신인 경우 자동으로 실행을 건너뜁니다.

---

#### 참고 예제 노트북

| 노트북 | 내용 |
|--------|------|
| `examples/100_using_ras_examples.ipynb` | 예제 프로젝트 사용법 |
| `examples/101_project_initialization.ipynb` | 프로젝트 초기화 |
| `examples/110_single_plan_execution.ipynb` | 단일 플랜 실행 |
| `examples/113_parallel_execution.ipynb` | 병렬 실행 |
| `examples/115_real_time_execution_monitoring.ipynb` | 실시간 모니터링 |

---

#### 자연어 프롬프트로 실행하기 (코드 없이)

Python 코드 없이 **Claude Code에 직접 말로 요청**해도 HEC-RAS를 실행할 수 있습니다.
내부적으로 `hecras-general-agent`가 자동으로 inspect → plan → execute → analyze 워크플로를 수행합니다.

**사용 가능한 프롬프트 예시:**

```
# 전체 워크플로 (가장 간단)
"C:/Projects/MyModel HEC-RAS 프로젝트를 실행해줘"
"이 프로젝트의 모든 플랜을 실행해줘"
"HEC-RAS 모델을 분석하고 실행해줘"

# 특정 플랜 지정
"플랜 01과 03만 실행해줘"
"plan 01을 C:/output 폴더에 결과를 저장해서 실행해줘"

# 검사만
"이 HEC-RAS 프로젝트에 어떤 플랜들이 있어?"
"프로젝트 구조를 분석해줘"

# 결과 분석
"실행된 플랜의 결과를 분석해줘"
"WSE 최댓값을 추출해줘"
```

**자동 실행 흐름:**

```
프롬프트 입력
    ↓
1. INSPECT  — 프로젝트 구조 분석, 실행 가능한 플랜 파악
    ↓
2. PLAN     — 최적 실행 모드 결정 (단일/병렬/원격)
    ↓
3. EXECUTE  — HEC-RAS 플랜 실행
    ↓
4. ANALYZE  — 결과 품질 검사, 이상치 탐지
    ↓
통합 워크플로 리포트 반환
```

**각 단계에서 Claude가 자동으로 하는 일:**

| 단계 | 담당 에이전트 | 하는 일 |
|------|-------------|---------|
| Inspect | `hecras-project-inspector` | `ras.plan_df` 분석, 실행 가능 여부 확인 |
| Plan | `hecras_plan_execution` skill | 플랜 수/리소스에 따라 단일/병렬 모드 결정 |
| Execute | `hecras_compute_plans` skill | `RasCmdr.compute_plan()` 또는 `compute_parallel()` 호출 |
| Analyze | `hecras-results-analyst` | 결과 HDF 파싱, PASS/WARN/FAIL 판정 |

**중단/재시작도 자연어로:**

```
"중단된 프로젝트 실행을 이어서 해줘"
→ 이미 완료된 플랜은 스킵하고 나머지만 실행

"결과가 맞는지 다시 실행해줘"
→ force_rerun=True 로 강제 재실행
```

> **핵심**: Python 코드를 직접 작성하지 않아도 됩니다. Claude Code에 목적을 말하면 적절한 에이전트와 스킬을 자동으로 선택하여 실행합니다.

### 2. HEC-RAS 결과 분석

- HDF5 파일에서 WSE, 유속, 수심 추출
- 정상류/비정상류 자동 감지
- Breach 결과 분석, 수리 특성 테이블 추출

### 3. 지오메트리 파싱 및 수정

- `.g##` 파일에서 단면 데이터 파싱 (고정폭 포맷)
- Manning's n 값 수정, 교량/암거/여수로 구조물 파싱
- 기하학 오류 자동 수리 (RasFixit)

### 4. 데이터 통합

- USGS NWIS에서 수위/유량 게이지 데이터 조회 및 경계조건 생성
- AORC 역사 강수 데이터, NOAA Atlas 14 설계홍수 처리
- HEC-DSS 파일 읽기/쓰기

### 5. FEMA eBFE/BLE 모델 처리

- 깨진 FEMA 배포 포맷을 자동으로 수정 (경로, Output/, Terrain/ 폴더)
- 41 GB 이상 대용량 모델도 처리

### 6. 노트북 및 문서 관리

- 예제 노트북 실행, 오류 감지, QA/QC
- mkdocs 문서 빌드 (GitHub Pages + ReadTheDocs)

---

## 현재 Claude Code 세팅 구조

### 계층적 지식 시스템 (`.claude/` 디렉토리)

```
.claude/
├── rules/          # 자동 로드되는 토픽별 규칙
│   ├── python/     # 코딩 패턴 (14개 파일)
│   ├── hec-ras/    # 도메인 지식 (7개 파일)
│   ├── testing/    # 테스트 방법론
│   ├── documentation/  # 문서 표준
│   └── validation/ # 검증 패턴
├── agents/         # 전문 서브에이전트 정의 (26개)
├── skills/         # 재사용 가능한 워크플로 스킬 (20개)
└── commands/       # 슬래시 명령어 (8개)
```

### 에이전트 3계층 아키텍처

| 계층 | 모델 | 역할 |
|------|------|------|
| **오케스트레이터** | Opus 4.6 | 고수준 계획, 서브에이전트 위임 |
| **전문가 서브에이전트** | Sonnet 4.6 | 도메인별 작업 (HDF, 지오메트리, USGS 등) |
| **작업 서브에이전트** | Haiku 4.5 | 단순 파일 읽기, 로그 검토, 패턴 매칭 |

### 주요 서브에이전트 (26개)

| 서브에이전트 | 역할 |
|---|---|
| `hecras-general-agent` | 전체 워크플로 조율 (inspect→execute→analyze) |
| `hecras-project-inspector` | 프로젝트 DataFrame 분석, 실행 준비도 점검 |
| `hecras-results-analyst` | 시뮬레이션 결과 해석, 이상치 탐지 |
| `hdf-analyst` | HDF5 파일 심층 분석 |
| `geometry-parser` | 지오메트리 파일 파싱 |
| `usgs-integrator` | USGS 게이지 데이터 통합 |
| `remote-executor` | 분산 원격 실행 |
| `precipitation-specialist` | 강수 데이터 처리 |
| `quality-assurance` | RasFixit 기반 품질 검사 |
| `code-oracle-codex` | OpenAI Codex로 심층 코드 분석 위임 |
| `code-oracle-gemini` | Google Gemini로 대용량 코드 검토 |
| `notebook-output-auditor` | 노트북 오류 검토 (Haiku) |

### 주요 Skills (20개)

| 카테고리 | 스킬 |
|---|---|
| **실행** | `hecras_compute_plans`, `hecras_compute_remote`, `hecras_compute_rascontrol` |
| **결과** | `hecras_extract_results`, `hecras_parse_compute-messages` |
| **파싱** | `hecras_parse_geometry`, `hecras_export_cloud-native` |
| **데이터** | `usgs_integrate_gauges`, `precip_analyze_aorc`, `dss_read_boundary-data` |
| **QA** | `qa_repair_geometry`, `qa_review_triple-model` |
| **개발도구** | `dev_invoke_codex-cli`, `dev_invoke_gemini-cli`, `dev_invoke_kimi-cli` |

### 슬래시 커맨드 (8개)

| 커맨드 | 용도 |
|---|---|
| `/agent-taskclose` | 세션 종료 시 정리 및 지식 보존 |
| `/agent-taskupdate` | 작업 진행상황 업데이트 |
| `/agent-cleanfiles` | 오래된 출력 파일 정리 |
| `/agent-engagesubagents` | 최적 서브에이전트 선택 |
| `/test-notebook` | 현재 노트북 실행 테스트 |
| `/agents-start-gitworktree` | 격리된 git worktree 생성 |
| `/agents-close-gitworktree` | worktree 정리 |
| `/agent-crossrepo` | ras-commander ↔ hms-commander 교차 작업 |

### 핵심 설계 원칙

1. **DataFrame-First**: 파일 경로는 항상 `ras.plan_df`, `ras.geom_df` 에서 가져옴
2. **Real Data Testing**: mock 없이 `RasExamples.extract_project()` 으로 실제 HEC-RAS 프로젝트 사용
3. **Lightweight Navigators**: 에이전트/스킬은 200-400줄의 안내자 역할 (중복 문서화 금지)
4. **Subagent Output Pattern**: 서브에이전트는 결과를 `.claude/outputs/` 에 마크다운 파일로 저장하고 경로를 반환
5. **Static Classes**: 대부분 클래스는 인스턴스화 없이 직접 호출 (`RasCmdr.compute_plan()`)

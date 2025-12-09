# 파이프라인 흐름

## 전체 파이프라인 개요

이 문서는 AI Startup Investment Evaluator의 전체 파이프라인 흐름을 단계별로 설명합니다.

## 파이프라인 단계별 상세 설명

### 1단계: 기업 탐색 및 선정 (startup_search_agent)

**입력**: 
- `input_text`: 사용자 입력 (예: "NextUnicorn에서 스타트업 50개 알려줘")

**처리 과정**:
1. **초기 크롤링**: NextUnicorn에서 스타트업 50개 크롤링
2. **태깅 및 RDB 저장**: 각 기업에 대해 LLM으로 태깅 수행 후 RDB에 저장
3. **후보 선정**: 태그별로 2~3개씩 후보 선정하여 최종 10개 기업 선정
4. **상세 크롤링**: 선정된 10개 기업에 대해 추가 크롤링 수행
5. **정제 및 VDB 저장**: 
   - 기업 소개/사업 내용/투자현황 등을 정제
   - Chunking → Embedding → VectorDB 임베딩 저장
   - 메타데이터: `type="company"`, `company_name`, `tags`

**출력**:
- `selected_companies`: 분석 대상 기업 리스트 (10개)
- RDB에 기업 태그 정보 저장
- VectorDB에 기업 상세 정보 임베딩 저장

**특징**:
- 2단계 크롤링으로 효율적인 데이터 수집
- 태그 기반 선정으로 다양성 확보
- RDB와 VectorDB의 역할 분리

---

### 2단계: 산업 탐색 및 벡터 DB 구축 (industry_search_agent)

**입력**: 
- `selected_companies`: 분석 대상 기업 리스트 (태그 정보 활용)

**처리 과정**:
1. **한경 컨센서스 PDF 수집**: 한경 컨센서스 PDF 수집 → 텍스트 변환
2. **Tavily Web Search**: 산업 관련 최신 데이터 수집
3. **Chunking 및 임베딩**: 수집된 데이터를 Chunking → Embedding
4. **VectorDB 저장**: 
   - 메타데이터: `type="industry"`, `tags` (산업 키워드)
   - 산업/시장 관련 대규모 벡터 DB 구축

**출력**:
- VectorDB에 산업 동향 정보 임베딩 저장
- 산업/시장 관련 대규모 벡터 DB 완성

**특징**:
- 다중 소스 데이터 수집 (PDF + Web Search)
- 대규모 벡터 DB 구축으로 풍부한 컨텍스트 확보

---

### 3단계: 기업별 순차 분석 시작 (resume_analysis)

**입력**: 
- `selected_companies`: 분석 대상 기업 리스트

**처리 과정**:
1. `selected_companies`에서 첫 번째 기업을 `pop(0)`으로 추출
2. `current_company` 설정
3. VectorDB에서 해당 기업의 태그 조회하여 `current_tags` 설정

**조건부 라우팅**:
- `current_company`가 있음 → `market_eval`로 이동
- `current_company`가 없음 + `report_written=True` → END
- `current_company`가 없음 + `report_written=False` → `report_writer`로 이동

**출력**:
- `current_company`: 현재 분석 중인 기업명
- `current_tags`: 현재 기업의 태깅 목록

**특징**:
- 순환 그래프 구조로 여러 기업을 순차 처리
- 상태 기반 조건부 라우팅

---

### 4단계: 시장성 평가 (market_eval_agent)

**입력**: 
- `current_company`: 현재 분석 중인 기업명
- `current_tags`: 현재 기업의 태깅 목록
- VectorDB의 산업/기업 정보

**처리 과정**:
1. **VDB 조회 - 산업 정보**: VectorDB에서 "산업 정보" 조회 (`type="industry"`)
   - 회사의 태그 기반으로 산업 트렌드/시장성/리스크 조회
2. **VDB 조회 - 기업 정보**: VectorDB에서 "기업 상세 정보" 조회 (`type="company"`)
3. **LLM 평가**: 두 정보를 LLM에 전달하여 시장성 평가 보고서 생성
4. **State 저장**: 결과 텍스트를 State의 `market_analysis`에 저장 후 반환

**출력**:
- `market_analysis`: 시장성 평가 보고서 (텍스트)

**특징**:
- RAG 활용으로 토큰 사용량 절감
- 태그 기반 산업 정보 검색으로 정확도 향상

---

### 5단계: 경쟁사 분석 (competitor_analysis_agent)

**입력**: 
- `current_company`: 현재 분석 중인 기업명
- `current_tags`: 현재 기업의 태깅 목록
- RDB의 기업 태그 정보

**처리 과정**:
1. **태그 읽기**: State에서 현재 기업의 태그(`current_tags`) 읽기
2. **RDB 조회**: RDB에서 동일 태그를 가진 "경쟁사 후보 기업" 조회
3. **유사 태그 그룹핑**: 유사 태그 기업들을 그룹핑
4. **LLM 분석**: 현재 기업 + 경쟁사 기업의 분석 내용을 LLM에 전달
5. **보고서 생성**: 경쟁 구도·차별점·위험요소 기반 보고서 생성
6. **State 저장**: 결과를 State의 `competitor_analysis`에 저장

**출력**:
- `competitor_analysis`: 경쟁사 분석 보고서 (텍스트)

**특징**:
- RDB 태그 기반 조회로 정확한 경쟁사 선정
- 태그 그룹 단위 비교로 데이터 일관성 확보

---

### 6단계: 투자 여부 판단 (investment_decision_agent)

**입력**: 
- `market_analysis`: 시장성 평가 결과
- `competitor_analysis`: 경쟁사 분석 결과
- `current_company`: 현재 분석 중인 기업명
- VectorDB (필요 시 추가 검색)

**처리 과정**:
1. **항목별 0/1 판단**: 특정 항목들에 대해 LLM에게 0/1 판단 요청
   - 예: 시장성, 제품 경쟁력, 팀 역량, 리스크 등
2. **추가 검색 (선택)**: 필요하면 VectorDB에서 수치 데이터/리포트 내용 보강 검색
3. **종합 판단**: 항목별 결과를 종합하여 투자 여부 True/False 결정
4. **State 저장**: State의 `investment_decision`에 저장

**출력**:
- `investment_decision`: 투자 여부 판단 결과 (True/False)

**조건부 라우팅**:
- `investment_decision=True` → `report_writer`로 이동
- `investment_decision=False` → `resume_analysis`로 이동 (다음 기업)

**특징**:
- 이진 판단(0/1) 기반 의사결정
- 필요 시 VectorDB 추가 검색으로 정확도 향상

---

### 7단계: 보고서 작성 (report_writer_agent)

**입력**: 
- `current_company`: 현재 분석 중인 기업명
- `market_analysis`: 시장성 평가 결과
- `competitor_analysis`: 경쟁사 분석 결과
- `investment_decision`: 투자 여부 판단 결과
- VectorDB (필요 시 추가 근거 검색)

**처리 과정**:
1. **State 조합**: State에 저장된 다음 정보를 조합
   - 시장성 분석 (`market_analysis`)
   - 경쟁사 분석 (`competitor_analysis`)
   - 투자 여부 판단 (`investment_decision`)
2. **추가 검색 (선택)**: 필요 시 VectorDB에서 추가 근거 검색
3. **보고서 생성**: 
   - LLM으로 보고서 본문 생성
   - 산업 분석과 기업 분석이 논리적으로 자연스럽게 이어지도록 구성
4. **PDF 저장**: 결과 PDF를 `/output/{company}.pdf`로 저장
5. **플래그 설정**: 하나라도 보고서를 생성하면 `report_written = True`

**출력**:
- PDF 보고서 파일 (`/output/{company}.pdf`)
- `report_written=True` 설정

**특징**:
- State 중심 + VectorDB 추가 검색으로 종합 정리
- 구조화된 보고서 템플릿으로 논리적 연결성 강화

---

### 8단계: 다음 기업 처리 또는 종료 (resume_analysis)

**입력**: 
- `selected_companies`: 남은 분석 대상 기업 리스트
- `report_written`: 보고서 작성 여부

**처리 과정**:
1. `selected_companies`에서 다음 기업 추출
2. 다음 기업이 있으면 → `market_eval`로 이동 (3단계로 돌아감)
3. 다음 기업이 없고 `report_written=True` → END
4. 다음 기업이 없고 `report_written=False` → `report_writer`로 이동

**특징**:
- 순환 그래프 구조로 여러 기업을 순차 처리
- 상태 기반 조건부 라우팅

---

## 파이프라인 흐름도 요약

```
[사용자 입력]
    ↓
[startup_search] → 스타트업 수집 및 구조화
    ↓
[industry_search] → 산업 동향 수집
    ↓
[resume_analysis] → 첫 번째 기업 추출
    ↓
[market_eval] → 시장성 평가
    ↓
[competitor_analysis] → 경쟁사 분석
    ↓
[investment_decision] → 투자 의사결정
    ↓
    ├─ 투자 승인 → [report_writer] → 보고서 생성
    │                                    ↓
    └─ 투자 거부 → [resume_analysis] → 다음 기업 또는 종료
```

## 파이프라인 특징

### 1. 순차 처리 구조
- 여러 기업을 순차적으로 평가
- 투자 거부 시 자동으로 다음 기업으로 전환

### 2. 상태 기반 데이터 전달
- 모든 노드가 AppState를 통해 데이터 공유
- 상태 불변성을 유지하여 부작용 최소화

### 3. 조건부 라우팅
- `resume_analysis`에서 기업 존재 여부에 따라 분기
- `investment_decision` 결과에 따라 보고서 생성 또는 다음 기업으로 이동

### 4. RAG 기반 정보 검색
- VectorDB에 저장된 정보를 유사도 검색으로 활용
- 메타데이터(`type`, `tags`) 기반 필터링으로 정확도 향상
- 토큰 사용량 절감 및 컨텍스트 확보

### 5. RDB와 VectorDB의 역할 분리
- **RDB**: 태그 정보 저장 및 태그 기반 경쟁사 조회
- **VectorDB**: 기업/산업 상세 정보 임베딩 저장 및 유사도 검색

## 성능 최적화 포인트

1. **2단계 크롤링**: 초기 50개 크롤링 후 태그 기반 선정으로 효율성 향상
2. **RDB 활용**: 태그 기반 경쟁사 조회 시 RDB 사용으로 빠른 조회
3. **RAG 활용**: 직접 LLM 호출 대신 RAG로 토큰 사용량 절감
4. **조건부 라우팅**: 투자 거부 시 즉시 다음 기업으로 전환하여 효율성 향상
5. **메타데이터 필터링**: `type`, `tags` 기반 필터링으로 검색 정확도 향상


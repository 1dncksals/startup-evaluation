# 시스템 아키텍처

## 개요

AI Startup Investment Evaluator는 LangGraph 기반의 멀티 에이전트 시스템으로, 각 에이전트가 특정 역할을 담당하며 상태 기반으로 정보를 공유하고 전달합니다.

## 시스템 구성 요소

### 1. 상태 관리 계층 (State Layer)

**AppState (TypedDict 기반)**
- `selected_companies`: 남은 분석 대상 기업 리스트 (처리 완료 시 pop)
- `current_company`: 현재 분석 중인 기업 (Optional[str])
- `current_tags`: 해당 기업의 태그 (List[str])
- `market_analysis`: 시장성 분석 결과 (Optional[str])
- `competitor_analysis`: 경쟁사 분석 결과 (Optional[str])
- `report_written`: 보고서 1개 이상 생성 여부 (bool)
- `investment_decision`: 투자 여부 판단 결과 True/False (Optional[bool])
- `input_text`: 사용자 입력 (예: "NextUnicorn에서 스타트업 2개 알려줘")

**설계 철학**:
- TypedDict를 사용하여 타입 안정성 확보
- Annotated 타입 힌트로 필드별 설명 추가하여 문서화 효과
- 각 노드에서 `dict(state)` 복사 후 업데이트하여 상태 불변성 유지

### 2. 에이전트 계층 (Agent Layer)

#### 2.1 기업 탐색 에이전트 (startup_search_agent)
**역할**: NextUnicorn 플랫폼에서 스타트업 정보 수집 및 선정

**주요 기능**:
1. **초기 크롤링**: NextUnicorn에서 스타트업 50개 크롤링
2. **태깅 및 RDB 저장**: 각 기업에 대해 LLM으로 태깅 수행 후 RDB(관계형 데이터베이스)에 저장
3. **후보 선정**: 태그별로 2~3개씩 후보 선정하여 최종 10개 기업 선정
4. **상세 크롤링**: 선정된 10개 기업에 대해 추가 크롤링 수행
5. **정제 및 VDB 저장**: 기업 소개/사업 내용/투자현황 등을 정제 → Chunking → VectorDB 임베딩 저장

#### 2.2 산업 탐색 에이전트 (industry_search_agent)
**역할**: 산업/시장 관련 대규모 벡터 DB 구축

**주요 기능**:
1. **한경 컨센서스 PDF 수집**: 한경 컨센서스 PDF 수집 → 텍스트 변환 → Chunking → VectorDB 저장
2. **Tavily Web Search**: 산업 관련 최신 데이터 수집 → Chunking → VectorDB 저장
3. **결과**: 산업/시장 관련 대규모 벡터 DB 구축

#### 2.3 시장성 평가 에이전트 (market_eval_agent)
**역할**: RAG를 활용하여 시장성 평가 보고서 생성

**주요 기능**:
1. **VDB 조회**: VectorDB에서 "산업 정보" 조회 (type="industry")
2. **기업 정보 조회**: VectorDB에서 "기업 상세 정보" 조회 (type="company")
3. **LLM 평가**: 두 정보를 LLM에 전달하여 시장성 평가 보고서 생성
4. **State 저장**: 결과 텍스트를 State의 `market_analysis`에 저장 후 반환

#### 2.4 경쟁사 분석 에이전트 (competitor_analysis_agent)
**역할**: 태그 기반 경쟁사 조회 및 분석 보고서 생성

**주요 기능**:
1. **태그 읽기**: State에서 현재 기업의 태그(`current_tags`) 읽기
2. **RDB 조회**: RDB에서 동일 태그를 가진 "경쟁사 후보 기업" 조회
3. **LLM 분석**: 현재 기업 + 경쟁사 기업의 분석 내용을 LLM에 전달
4. **보고서 생성**: 경쟁 구도·차별점·위험요소 기반 보고서 생성
5. **State 저장**: 결과를 State의 `competitor_analysis`에 저장

#### 2.5 투자 여부 판단 에이전트 (investment_decision_agent)
**역할**: 항목별 0/1 판단을 종합하여 투자 여부 결정

**주요 기능**:
1. **항목별 판단**: 특정 항목들에 대해 LLM에게 0/1 판단 요청
   - 예: 시장성, 제품 경쟁력, 팀 역량, 리스크 등
2. **종합 판단**: 항목별 결과를 종합하여 투자 여부 True/False 결정
3. **State 저장**: State의 `investment_decision`에 저장

#### 2.6 보고서 작성 에이전트 (report_writer_agent)
**역할**: State에 저장된 분석 결과를 조합하여 보고서 생성

**주요 기능**:
1. **State 조합**: State에 저장된 다음 정보를 조합
   - 시장성 분석 (`market_analysis`)
   - 경쟁사 분석 (`competitor_analysis`)
   - 투자 여부 판단 (`investment_decision`)
2. **추가 검색**: 필요 시 VectorDB에서 추가 근거 검색
3. **보고서 생성**: 결과 PDF를 `/output/{company}.pdf`로 저장
4. **플래그 설정**: 하나라도 보고서를 생성하면 `report_written = True`

### 3. 데이터 계층 (Data Layer)

#### 3.1 Chroma VectorDB
**역할**: 기업 및 산업 데이터 임베딩 저장 및 유사도 검색

**저장 데이터**:
- 기업 정보 (type="company", 메타데이터: company_name, tags)
- 산업 동향 정보 (type="industry", 메타데이터: tags)

**검색 전략**:
- 유사도 기반 검색으로 컨텍스트 확보
- 메타데이터 필터링으로 정확도 향상

#### 3.2 관계형 데이터베이스 (RDB)
**역할**: 기업 태그 정보 저장 및 태그 기반 경쟁사 조회

**저장 데이터**:
- 기업명 및 태그 정보
- 태그별 기업 그룹핑 정보

**사용 목적**:
- 태그별 후보 선정 (기업 탐색 에이전트)
- 동일 태그 기반 경쟁사 조회 (경쟁사 분석 에이전트)

#### 3.3 외부 데이터 소스
- **NextUnicorn**: 스타트업 정보 크롤링
- **한경 컨센서스**: 산업 동향 PDF 자료
- **Tavily Web Search API**: 산업 동향 실시간 수집

### 4. 워크플로우 오케스트레이션 계층 (Orchestration Layer)

#### 4.1 LangGraph 그래프 구조
**노드 구성**:
- `startup_search`: 스타트업 검색 및 선정
- `industry_search`: 산업 동향 수집
- `resume_analysis`: 다음 기업으로 전환 또는 종료
- `market_eval`: 시장성 평가
- `competitor_analysis`: 경쟁사 분석
- `investment_decision`: 투자 의사결정
- `report_writer`: 보고서 생성

**조건부 라우팅**:
- `resume_analysis`에서 `current_company` 존재 여부에 따라 분기
- `investment_decision` 결과에 따라 보고서 생성 또는 다음 기업으로 이동

#### 4.2 순환 그래프 설계
- `report_writer → resume_analysis`로 연결하여 여러 기업을 순차 처리
- 투자 거부 시 자동으로 다음 기업으로 전환

## 데이터 흐름

### 입력 → 처리 → 출력

1. **입력**: 사용자 입력 (`input_text`: "NextUnicorn에서 스타트업 20개 알려줘")
2. **처리**: 
   - 스타트업 수집 및 구조화
   - 산업 동향 수집
   - 기업별 순차 분석 (시장성 → 경쟁사 → 투자 의사결정)
3. **출력**: 
   - 투자 승인 기업: `{company_name}_investment_report.pdf` (레이더 차트 포함)
   - 전체 거부: `comprehensive_rejection_report.pdf`

## 설계 원칙

### 1. 상태 기반 데이터 전달
- 모든 노드는 AppState를 통해 데이터를 공유
- 상태 불변성을 유지하여 부작용 최소화

### 2. 역할 분리
- 각 에이전트는 단일 책임 원칙을 따름
- 명확한 역할 분리로 유지보수성 향상

### 3. 확장 가능성
- 도메인별 확장 가능한 구조
- 평가 기준 외부화 가능

### 4. 비용 효율성
- RAG 활용으로 LLM 토큰 사용량 절감
- VectorDB 캐싱으로 중복 크롤링 방지

## RAG 전략

### 메타데이터 구성

시스템은 VectorDB에 저장된 데이터를 효율적으로 검색하기 위해 메타데이터를 활용합니다.

**메타데이터 필드**:
- `type`: "company" 또는 "industry" (데이터 유형 구분)
- `company_name`: 기업명 (기업 데이터일 경우)
- `tags`: 태그 리스트 (기업 태그 또는 산업 키워드)

### Indexing Stage (인덱싱 단계)

#### 1. 기업 탐색 데이터 인덱싱
**데이터 소스**: NextUnicorn 크롤링 결과

**처리 과정**:
1. 크롤링한 기업 정보를 정제 (기업 소개/사업 내용/투자현황 등)
2. Chunking: 긴 텍스트를 의미 있는 단위로 분할
3. Embedding: BAAI/bge-m3 모델로 벡터 임베딩 생성
4. VectorDB 저장: Chroma VectorDB에 저장

**메타데이터**:
- `type="company"`
- `company_name`: 기업명
- `tags`: LLM이 생성한 태그 리스트

#### 2. 산업 탐색 데이터 인덱싱
**데이터 소스**: 한경 컨센서스 PDF, Tavily Web Search 데이터

**처리 과정**:
1. 한경 컨센서스 PDF → 텍스트 변환
2. Tavily Web Search로 산업 관련 최신 데이터 수집
3. Chunking: 긴 텍스트를 의미 있는 단위로 분할
4. Embedding: BAAI/bge-m3 모델로 벡터 임베딩 생성
5. VectorDB 저장: Chroma VectorDB에 저장

**메타데이터**:
- `type="industry"`
- `tags`: 산업 키워드 (예: "전기차", "전동킥보드", "자율주행")

**결과**: 산업/시장 관련 대규모 벡터 DB 구축

### Retrieval Stage (검색 단계)

#### 1. 시장성 평가
**검색 전략**:
- `type="industry"` 기반 컨텍스트 검색
- 회사의 태그(`current_tags`) 기반으로 산업 트렌드/시장성/리스크 조회
- 기업 상세 정보도 추가 조회 (`type="company"`)

**활용 방법**:
- VectorDB에서 유사도 검색으로 관련 산업 정보 수집
- 태그 기반 필터링으로 정확도 향상
- 수집된 컨텍스트를 LLM에 전달하여 시장성 평가 보고서 생성

#### 2. 경쟁사 분석
**검색 전략**:
- RDB에서 태그 필터 기반 기업 검색
- 동일 태그를 가진 경쟁사 후보 기업 조회
- 유사 태그 기업들을 그룹핑 후 LLM 비교분석

**활용 방법**:
- RDB의 빠른 태그 기반 조회로 경쟁사 후보 선정
- VectorDB에서 선정된 경쟁사의 상세 정보 조회
- 현재 기업 + 경쟁사 정보를 LLM에 전달하여 경쟁 구도·차별점·위험요소 분석

#### 3. 투자 여부 판단
**검색 전략**:
- 시장성/경쟁사 분석 결과를 기반으로 LLM에서 판단
- 필요하면 VectorDB에서 수치 데이터/리포트 내용 보강 검색

**활용 방법**:
- State에 저장된 분석 결과를 종합
- 추가 정보가 필요한 경우 VectorDB에서 보강 검색
- 항목별 0/1 판단을 종합하여 투자 여부 결정

#### 4. 보고서 작성
**검색 전략**:
- State 중심 + 필요한 경우 VectorDB에서 추가 근거 검색하여 종합 정리

**활용 방법**:
- State에 저장된 모든 분석 결과 조합
- 논리적 연결이 필요한 경우 VectorDB에서 추가 근거 검색
- 산업 분석과 기업 분석이 자연스럽게 이어지도록 보고서 생성

## 기술적 특징

### RDB와 VectorDB의 역할 분리
- **RDB**: 태그 정보 저장 및 태그 기반 경쟁사 조회 (빠른 조회)
- **VectorDB**: 기업/산업 상세 정보 임베딩 저장 및 유사도 검색 (의미 기반 검색)

### 하이브리드 접근
- **태그 기반 조회**: RDB를 활용한 빠른 경쟁사 선정
- **의미 기반 검색**: VectorDB를 활용한 컨텍스트 확보
- **직접 LLM 호출**: 구조화 및 태깅 등 정확도가 중요한 작업

## 확장 가능성

이 아키텍처는 다음과 같이 확장 가능합니다:

1. **도메인 확장**: 모빌리티 외 헬스케어, 핀테크 등으로 확장 (산업별 검색 쿼리만 변경)
2. **평가 기준 커스터마이징**: YAML/JSON 설정 파일로 평가 기준 및 가중치 관리
3. **비동기 병렬 처리**: 독립적인 에이전트를 병렬 실행하여 성능 향상
4. **실시간 모니터링**: LangSmith 등으로 각 노드 실행 시간, LLM 호출 비용, 에러율 모니터링


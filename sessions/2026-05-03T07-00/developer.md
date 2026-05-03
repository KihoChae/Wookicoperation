# 💻 Developer — Business가 정의한 Task Breakdown을 기반으로, 이 작업을 성공적으로 수행하기 위해 필요한 '자동화 로직의 우선순위'와 '기술 스택(Pseudo-Code 필요 영역)'을 구체적으로 설계하여 기술적 실행 계획안을 작성하십시오.

💻 Developer: 작업 시작합니다.

# ⚙️ 기술적 실행 계획안 (Technical Implementation Plan) - MVP Validation 파이프라인 구축

## 🎯 목표 재확인 및 우선순위 조정
Secretary와 Business가 정의한 핵심 액션은 **'A/B 테스트용 미니 웹툰 시나리오(3컷) 2종 제작 $\rightarrow$ 커뮤니티 배포 $\rightarrow$ KPI 측정'** 입니다.
이 과정 중, 현재 가장 기술적 난이도가 높고 파이프라인 자동화의 핵심이 될 부분은 **"수집된 비정형 텍스트 데이터(댓글)를 구조화하고 KPI에 따라 점수화하는 백엔드 로직"**입니다.

따라서 이번 사이클의 최우선 개발 목표는, 향후 수동으로 데이터를 취합할 시간을 최소화하기 위해 **'반복적인 데이터 전처리 및 분석을 자동화하는 모듈'** 설계입니다.

## 🚀 기술 스택 및 아키텍처 제안
*   **주요 언어:** Python (데이터 처리 라이브러리 활용 용이)
*   **핵심 기술:** LLM API Wrapper, Web Scraping/API Connector, NLP Model (Sentiment/Topic Modeling)
*   **아키텍처 패턴:** **Data Ingestion $\rightarrow$ Data Cleaning $\rightarrow$ Feature Extraction (KPI Scoring) $\rightarrow$ Reporting**

## 🛠️ 자동화 로직 우선순위 및 Pseudo-Code 설계

### 🥇 Priority 1: 댓글 데이터 수집 및 정규화 모듈 (`data_ingestion.py`)
*   **기능:** 타겟 커뮤니티(예: 네이버 카페, 특정 Reddit 서브레딧 등)의 댓글 데이터를 주기적으로 크롤링하거나 API를 통해 가져옵니다. (실제 환경에서는 각 플랫폼별 인증 및 방어 로직이 필요함.)
*   **기술 스택:** `requests`, `BeautifulSoup` 또는 공식 SDK.
*   **Pseudo-Code 영역:**

```python
# data_ingestion.py - Pseudo Code for Comment Scraping/API Call
FUNCTION fetch_comments(source: str, target_url: str) -> List[Dict]:
    """지정된 소스에서 댓글 데이터를 가져와 표준화된 딕셔너리 리스트로 반환한다."""
    try:
        # API Key 관리 필요 (Environment Variable 사용 권장)
        response = make_api_call(source, target_url, api_key=os.getenv("COMMUNITY_API_KEY"))
        raw_data = parse_response(response) # 플랫폼별 파싱 로직 추가 필요

        standardized_comments = []
        FOR item IN raw_data:
            comment = {
                "user_id": item.get("author"),
                "timestamp": item.get("date"),
                "raw_text": item.get("content", ""), # 핵심 텍스트
                "source": source
            }
            standardized_comments.append(comment)
        return standardized_comments

    EXCEPT ConnectionError:
        LOG_ERROR("데이터 수집 실패: 연결 오류 발생.")
        RETURN []
```

### 🥈 Priority 2: KPI Feature Extraction 및 Scoring 엔진 (`kpi_scorer.py`)
*   **기능:** `raw_text`를 입력받아, Business가 정의한 세 가지 KPI(PIR, ISR, IGR)에 대한 점수화 로직을 실행합니다. **이 부분이 LLM API 호출이 가장 많이 사용될 영역입니다.**
*   **기술 스택:** Python + LLM SDK (OpenAI/Claude 등).
*   **Pseudo-Code 영역:**

```python
# kpi_scorer.py - Pseudo Code for NLP Analysis using LLM
FUNCTION score_comment(comment: Dict, prompt_template: str) -> Dict:
    """LLM을 호출하여 텍스트에 대한 다차원적 분석 점수를 얻는다."""
    context = f"웹툰 시나리오 A/B 테스트 댓글. 당신은 전문 데이터 분석가입니다. 다음 댓글을 분석하고 JSON 형식으로 출력하세요.\n\n댓글: {comment['raw_text']}"

    # LLM 호출 (Temperature는 0.1 정도로 낮게 설정하여 일관성 유지)
    llm_response = call_llm_api(context, system_prompt="You are a structured data analyst.")

    # JSON 파싱 및 검증 로직 추가 필수
    try:
        analysis_json = json.loads(llm_response['text'])
        return {
            "PIR_score": analysis_json.get("PIR", 0), # Problem Identification Ratio Score (0-5)
            "ISR_score": analysis_json.get("ISR", 0), # Interest/Self-Relatability Score (0-5)
            "IGR_score": analysis_json.get("IGR", 0)  # Information Gain Request Score (0-5)
        }
    EXCEPT json.JSONDecodeError:
        LOG_WARNING(f"LLM 응답 파싱 실패: {comment['raw_text']}")
        RETURN {"PIR_score": None, "ISR_score": None, "IGR_score": None}

FUNCTION calculate_final_kpis(scored_comments: List[Dict]) -> Dict:
    """모든 댓글 점수를 집계하여 최종 비율을 계산한다."""
    # ... (총합산 및 평균 계산 로직)
    pass
```

### 🥉 Priority 3: 통합 리포팅 및 시각화 (`report_generator.py`)
*   **기능:** 모든 데이터를 취합하고, Business가 원하는 형식(KPI 대시보드)으로 보고서를 자동 생성합니다. (Markdown 또는 Plotly 그래프).
*   **기술 스택:** `pandas`, `matplotlib`/`plotly`.

## 💡 개발 우선순위 및 실행 계획 요약

1.  **[개발]**: Priority 1 모듈을 **Mock Data 기반**으로 먼저 완성하여, 실제 데이터 수집 없이도 전체 파이프라인의 흐름(Input $\rightarrow$ Output)을 테스트할 수 있도록 한다.
2.  **[검토 요청]**: 이 기술 설계안과 Pseudo-Code를 검토한 후, **실제 타겟 커뮤니티의 API 접근 방식 또는 Mock Data 예시**가 필요하다.

---
📊 평가: 진행중 — 핵심적인 자동화 로직(데이터 수집 $\rightarrow$ NLP 분석)의 아키텍처 설계 및 기술적 난이도에 따른 우선순위를 명확히 했음. 실제 구현을 위해 데이터 소스 정의와 LLM 프롬프트 튜닝이 다음 단계 필요함.
📝 다음 단계: Business에게 Priority 1 모듈 테스트를 위한 '가상 커뮤니티 댓글 데이터셋(Mock JSON)' 예시를 요청하거나, 혹은 Mock Data를 활용하여 Priority 2의 LLM 호출 로직에 대한 구체적인 프롬프트 엔지니어링을 진행해야 함.

## 샘플문제 실행 및 벡터스토어 적용 안내

### 1. 샘플문제 실행 방법

- **"(난이도표시본) 샘플문제 워드로 다시.docx"** 파일을 다운로드하여 전체 코드를 실행하세요.

---

**OR**

---

### 2. 벡터스토어 적용 방법

1. **벡터스토어 다운로드**
   - 벡터스토어 파일을 다운로드합니다.

2. **파일 위치 설정**
   - 다운로드한 벡터스토어를  
     `"ai튜터(테스트).ipynb"`의 마지막 코드와 동일한 경로(예: Google Drive)에 위치시킵니다.

3. **벡터스토어 경로 설정**
   ```python
   store_dir = '/content/drive/MyDrive/tutor_store'
   faiss_path = os.path.join(store_dir, "medical_qa_index.faiss")
   chunk_path = os.path.join(store_dir, "chunk_mapping.pkl")

---
# 📑 전체 로직 요약

1) Google Drive에서 FAISS 인덱스와 문제 청크를 불러와 벡터 검색 기반 RAG 체인을 구성하고,  
2) `SessionState`로 사용자별 난이도·정답률을 관리하며,  
3) 커스텀 분할기와 Fake 임베딩으로 문제를 준비한 뒤,  
4) LangChain 프롬프트 템플릿으로 출제·피드백 로직을 연결하고,  
5) Gradio 버튼·채팅 컴포넌트에 답안 처리·피드백·다음 문제 제시 기능을 바인딩하여 대화형 UI를 완성합니다.

---

## 1. 라이브러리 및 환경 설정  
- **Colab 전용**: `drive.mount`, `userdata.get`으로 Drive 마운트 및 비밀 API 키 로드  
- **벡터 연산**: `faiss`, `numpy`  
- **웹 UI**: `gradio`  
- **LangChain 핵심**: FAISS 벡터스토어, `ChatPromptTemplate`, Google Gemini LLM, 문서·체인 관련 클래스  

## 2. 벡터스토어 파일 경로 정의  
- `medical_qa_index.faiss`와 `chunk_mapping.pkl` 파일 경로를 `/content/drive/MyDrive/tutor_store` 아래에 설정  

## 3. SessionState 클래스  
- **속성**: 현재 난이도(`난이도 하/중/상`), 정답·오답 카운트, 현재 문제·정답 저장, 본 문제 ID 집합  
- **메서드**:  
  - `total_attempts`, `success_rate` 프로퍼티로 시도 횟수와 성공률 계산  
  - `update_stats(is_correct)`로 정답 여부에 따라 카운트 갱신 및 난이도 자동 상·하향  

## 4. 문제 분할기 & 임베딩  
### 4.1 MedicalQuestionSplitter  
- 정규표현식 패턴(“난이도 하 1.” 등)으로 텍스트를 문제 단위 청크로 분리  
### 4.2 CustomFakeEmbeddings  
- 실제 임베딩 대신 모두 0벡터를 반환해 FAISS 인덱스와 호환  

## 5. 벡터스토어 로드 함수 (`load_vectorstore`)  
1. FAISS 인덱스 파일을 읽고  
2. pickle로 저장된 청크 리스트를 `Document` 객체로 변환(메타데이터에 레벨 포함)  
3. Fake 임베딩 기반 LangChain FAISS 인스턴스 생성  
4. InMemoryDocstore에 문서 매핑 후 반환  

## 6. 유틸리티 함수들  
- `get_level_from_text(text)`: 청크 앞의 “난이도” 정보 추출  
- `process_question(text)`: 본문에서 문제·정답·해설 분리  
- `hide_answer_from_question(text)`: 문제만 남기고 정답 숨기기  

## 7. RAG 체인 및 리트리버 설정 (`setup_rag_chain`)  
- Google Gemini LLM(“gemini-1.5-pro”) 인스턴스 생성  
- 로드된 벡터스토어로부터 “유사도 검색(k=3)” 리트리버 생성  
- LLM · 리트리버 · 문서 리스트 반환  

## 8. 커스텀 질문 선택기 (`create_question_selector`)  
- 세션의 `current_level`과 본 문제 ID(`question_ids`)를 기준으로  
- 아직 풀지 않은 동일 난이도 문서 중 무작위 선택  
- 소진 시 다른 난이도로 이동하거나 IDs 초기화 후 재출제  

## 9. 프롬프트 템플릿 정의  
- **출제용**: 문제만 학생에게 전달 (`question_template`)  
- **피드백용**: 시스템·휴먼 메시지로 답안 평가 및 피드백 지침 제공 (`feedback_template`)  

## 10. 피드백 체인 캐싱 (`get_feedback_chain`)  
- 최초 호출 시에만 `RunnablePassthrough → feedback_template → LLM → StrOutputParser` 체인 생성  
- 이후 재사용으로 성능 최적화  

## 11. 문제 흐름 관리  
- `select_next_question()`: CustomRetriever에서 다음 `Document` 반환  
- `prepare_next_question()`: 학습 현황(난이도·정답률·문제 수) 문자열 + 숨겨진 문제 텍스트 반환  
- `generate_feedback()`: 학생 답안과 정답 여부를 LLM 피드백 체인에 전달, 예외 시 기본 메시지, 오답 시 정답·해설 추가  

## 12. Gradio UI 구성 및 실행  
- `chat_interface()`:  
  1. 첫 메시지 시 인사 + 첫 문제 제시  
  2. 이후 답안 평가 → `update_stats` → `generate_feedback` → `prepare_next_question` 순으로 대화 내역 갱신  
- 버튼 클릭 핸들러: `answer_click(A/B/C/D)`, `clear_chat()`  
- `Blocks`로 제목·채팅창·버튼 배치, `demo.launch(share=True)`로 외부 공유  

---


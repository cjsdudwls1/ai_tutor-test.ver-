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

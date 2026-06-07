# 음식 레시피 추천

## 📁 최종 폴더 구조

```
Food_recipe_project/
├── 📂 src/                          # ⭐ 메인 소스 코드 디렉토리
│   ├── __init__.py                  # 패키지 초기화
│   ├── config.py                    # 🔧 설정 파일 (API 키, 경로, 파라미터)
│   ├── embeddings.py                # 📝 텍스트 임베딩 (OpenAI)
│   ├── image_detection.py           # 🖼️ YOLO 이미지 감지
│   ├── translation.py               # 🔤 GPT 재료명 번역 (캐싱)
│   ├── recipe_search.py             # 🔍 FAISS 레시피 검색
│   └── main.py                      # 🚀 메인 실행 파일 (통합)
│
├── 📂 models/                       # 🤖 훈련된 모델 파일
│   └── best.pt                      # YOLO 이미지 분류 모델
│
├── 📂 data/                         # 💾 데이터 파일
│   ├── recipes_faiss.index          # FAISS 벡터 인덱스
│   └── recipes_meta.json            # 레시피 메타데이터 JSON
│
├── requirements.txt                 # 📦 Python 의존성
├── README.md                        # 📖 프로젝트 설명서
├── .env.example                     # 환경 변수 예시 파일
├── .gitignore                       # 민감한 파일 제외 설정 (.env 포함)
└── ARCHITECTURE.md                  # 이 파일
```

---

## 🏗️ 아키텍처 설명

### 데이터 흐름

```
🖼️ 이미지 입력
    ↓
[image_detection.py - YOLO]
    ↓
📋 감지된 재료 (영어)
    ↓
[translation.py - GPT]
    ↓
🔤 번역된 재료 (한국어)
    ↓
[embeddings.py - OpenAI]
    ↓
📊 재료 벡터 임베딩
    ↓
[recipe_search.py - FAISS]
    ↓
🍳 추천 레시피 반환
```

---

## 📚 각 모듈 상세 설명

### 1️⃣ `config.py` - 프로젝트 설정
**역할:** 모든 설정을 중앙화하여 관리 (python-dotenv로 `.env` 자동 로드)
```python
# 제공하는 설정
- OPENAI_API_KEY          # OpenAI API 키
- EMBEDDING_MODEL         # 임베딩 모델 (text-embedding-3-small)
- GPT_MODEL              # GPT 모델 (gpt-4o-mini)
- FAISS_INDEX_PATH       # FAISS 인덱스 경로
- RECIPES_META_PATH      # 레시피 메타 JSON 경로
- YOLO_MODEL_PATH        # YOLO 모델 경로
- YOLO_CONF/IOU          # YOLO 감지 파라미터
```
설정 로드:
```python
from dotenv import load_dotenv
load_dotenv(PROJECT_ROOT / ".env")  # 루트의 .env 자동 로드
OPENAI_API_KEY = os.getenv("OPENAI_API_KEY")
```

### 2️⃣ `embeddings.py` - 임베딩 처리
**역할:** 텍스트를 벡터로 변환
```python
embed_text(text) → list[float]
```
- OpenAI의 `text-embedding-3-small` 모델 사용
- 재료 검색에 사용되는 벡터화

### 3️⃣ `image_detection.py` - YOLO 이미지 감지
**역할:** 이미지에서 재료 자동 감지
```python
IngredientDetector 클래스
  └─ detect_from_image(image_path) → list[str]
```
- YOLO 모델로 재료 탐지
- 중복 제거된 클래스명 반환

### 4️⃣ `translation.py` - GPT 번역
**역할:** 영어 재료명을 한국어로 변환
```python
translate_ingredient_en2ko(label) → str     # 캐싱됨
translate_ingredients_list_to_korean(list) → list
```
- LRU 캐시로 성능 최적화
- 반복되는 번역 요청 빠르게 처리

### 5️⃣ `recipe_search.py` - FAISS 검색
**역할:** 재료 기반 유사 레시피 검색
```python
RecipeSearchEngine 클래스
  ├─ search_by_ingredients(재료) → list[dict]
  └─ print_results(결과) → None
```
- FAISS 벡터 유사도 검색
- 메타데이터 기반 필터링

### 6️⃣ `main.py` - 통합 실행 파일
**역할:** 전체 파이프라인 통합
```python
search_recipes_by_image(image_path) → None
search_recipes_by_text(ingredients) → None
```
- 이미지 기반 검색: 감지 → 번역 → 검색
- 텍스트 기반 검색: 직접 레시피 검색
 - 기본 실행 시: 프로젝트 루트의 `test_ingredients.png`로 자동 검색

---

## 🔄 모듈 간 의존성

```
config.py (설정)
    ↑
    ├── embeddings.py ──────┬─→ recipe_search.py
    │                       └─→ main.py
    │
    ├── image_detection.py ─────→ main.py
    │
    ├── translation.py ──────────→ main.py
    │
    └── recipe_search.py ────────→ main.py
```

---

## 💡 사용 예시

### 텍스트 기반 검색
```python
from src.main import search_recipes_by_text

search_recipes_by_text("당근, 양파, 감자", k=10)
```

### 이미지 기반 검색
```python
from src.main import search_recipes_by_image

search_recipes_by_image("path/to/image.jpg", k=10)
```

### 커맨드라인
```bash
cd src
python main.py                 # 기본: test_ingredients.png 이미지로 검색
python main.py /path/img.png   # 특정 이미지로 검색
python main.py "당근 양파 감자"  # 텍스트로 재료 검색
```

---

## 🔐 보안 고려사항

⚠️ **API 키 관리**
- `.env` 파일에 `OPENAI_API_KEY` 저장, git에 커밋 금지 (`.gitignore` 적용)
- `python-dotenv`로 `.env` 자동 로드 (이미 `requirements.txt`에 포함)
```bash
echo "OPENAI_API_KEY=your-key-here" > .env
```

💻 **macOS 실행 이슈 (OpenMP)**
- 드물게 OpenMP 중복 초기화 오류가 발생할 수 있습니다.
- 임시 해결: 환경 변수 설정 후 실행
```bash
export KMP_DUPLICATE_LIB_OK=TRUE
python main.py
```

---

작성일: 2026-06-07

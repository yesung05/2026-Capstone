---
name: quiz-generator
description: >
  PDF 강의자료를 업로드하면 핵심 내용을 추출하여 시험 준비용 퀴즈 30개를 자동 생성하는 스킬.
  퀴즈는 객관식(5%), 단답형(85%), OX(10%) 비율로 구성되며, 상호작용 가능한 JSX 퀴즈 앱과 Anki 호환 CSV 다운로드 기능을 함께 제공한다.
  사용자가 "강의자료", "퀴즈 만들어줘", "시험 준비", "문제 만들어줘", "공부 자료", "복습용 퀴즈", "Anki", "플래시카드" 등을 언급하거나 PDF를 업로드하며 퀴즈 생성을 요청할 때 반드시 이 스킬을 사용하라.
  파일이 업로드되지 않아도 텍스트 형태로 내용을 붙여넣으면 동일하게 동작한다.
---

# Quiz Generator Skill

강의자료에서 핵심 내용을 추출하고 시험 준비용 퀴즈를 생성하는 스킬.

## 전체 워크플로우

1. **파일 읽기** → 2. **핵심 내용 추출** → 3. **퀴즈 JSON 생성** → 4. **JSX 퀴즈 앱 생성** → 5. **CSV 다운로드 포함**

---

## Step 1: 파일 읽기

**주 지원 형식: PDF** (PPTX, DOCX, 텍스트도 지원)

실제 파일 경로는 먼저 `ls /mnt/user-data/uploads/` 로 확인 후 사용한다.

### PDF (기본)
```bash
pip install pdfplumber --break-system-packages -q
python3 << 'EOF'
import pdfplumber

path = '/mnt/user-data/uploads/파일명.pdf'
with pdfplumber.open(path) as pdf:
    chunks = []
    for i, page in enumerate(pdf.pages):
        t = page.extract_text() or ''
        if t.strip():
            chunks.append(f"[페이지 {i+1}]\n{t}")
    full_text = '\n\n'.join(chunks)

# 12000자 초과 시: 앞 5000 + 중간 4000 + 끝 3000 샘플링 (30문제 생성에 충분)
if len(full_text) > 12000:
    mid = len(full_text) // 2
    full_text = full_text[:5000] + '\n...(중략)...\n' + full_text[mid-2000:mid+2000] + '\n...(중략)...\n' + full_text[-3000:]

print(full_text)
EOF
```

### PPTX (보조)
```bash
pip install python-pptx --break-system-packages -q
python3 -c "
from pptx import Presentation
prs = Presentation('/mnt/user-data/uploads/파일명.pptx')
text = []
for i, slide in enumerate(prs.slides):
    parts = [f'[슬라이드 {i+1}]']
    for shape in slide.shapes:
        if hasattr(shape, 'text') and shape.text.strip():
            parts.append(shape.text)
    text.append('\n'.join(parts))
print('\n\n'.join(text)[:12000])
"
```

### Word / 텍스트 (보조)
```bash
pip install python-docx --break-system-packages -q
python3 -c "
from docx import Document
doc = Document('/mnt/user-data/uploads/파일명.docx')
text = '\n'.join(p.text for p in doc.paragraphs if p.text.strip())
print(text[:12000])
"
```

---

## Step 2: 퀴즈 생성 규칙

총 30개 기준:
- **객관식 (5%)**: 2개 — 4지선다, 핵심 개념/정의
- **단답형 (85%)**: 25개 — 1~3단어 또는 짧은 문장으로 답하는 문제
- **OX (10%)**: 3개 — 참/거짓 판별 문제

### 퀴즈 생성 프롬프트 (Anthropic API 사용)

```javascript
const systemPrompt = `당신은 교육 전문가입니다. 제공된 강의 내용에서 핵심 개념을 추출하여 시험 준비용 퀴즈를 생성합니다.

반드시 아래 JSON 형식만 반환하세요 (마크다운 코드블록 없이):
{
  "title": "강의 제목 또는 주제",
  "questions": [
    {
      "id": 1,
      "type": "multiple_choice",
      "question": "문제 텍스트",
      "options": ["A. 선택지1", "B. 선택지2", "C. 선택지3", "D. 선택지4"],
      "answer": "A",
      "explanation": "해설"
    },
    {
      "id": 2,
      "type": "short_answer",
      "question": "문제 텍스트",
      "answer": "정답",
      "explanation": "해설"
    },
    {
      "id": 3,
      "type": "ox",
      "question": "문제 텍스트",
      "answer": "O",
      "explanation": "해설"
    }
  ]
}

비율: 객관식 2개, 단답형 25개, OX 3개 (총 30개)
- 객관식: type = "multiple_choice", answer는 "A"/"B"/"C"/"D"
- 단답형: type = "short_answer", answer는 짧고 명확한 답 (1~3단어 권장)
- OX: type = "ox", answer는 "O" 또는 "X"
- 내용을 고르게 분산하여 강의 전반을 커버하도록 출제할 것`;
```

---

## Step 3: JSX 퀴즈 앱 생성

퀴즈 JSON이 생성되면 아래 구조의 JSX React 컴포넌트를 생성한다.

### 필수 기능
- **진행바**: 현재 문제 번호 / 전체 문제 수
- **문제 유형 배지**: 객관식 / 단답형 / OX 표시
- **입력 방식**:
  - 객관식: 라디오 버튼 또는 클릭 가능한 선택지 카드
  - 단답형: 텍스트 입력 필드 (대소문자, 띄어쓰기 무시 채점)
  - OX: O / X 버튼
- **즉시 피드백**: 정답/오답 표시 + 해설
- **결과 화면**: 점수, 정답률, 틀린 문제 복습
- **CSV 다운로드 버튼**: 결과 화면 또는 상단에 항상 표시

### Anki 호환 CSV 다운로드 형식

Anki에서 바로 임포트할 수 있는 형식으로 출력한다.

**Anki 임포트 설정 안내 (앱 내 표시):**
- 파일 형식: CSV (UTF-8)
- 구분자: 쉼표
- 필드 순서: Front → Back → Tags
- 노트 타입: Basic (앞/뒤)

```
Front,Back,Tags
"미토콘드리아의 주요 기능은?","ATP(에너지) 생산 — 산화적 인산화를 통해 세포 에너지를 공급한다.","세포생물학 단답형"
"세포막은 무엇으로 구성되어 있는가?<br><br>A. 인지질 이중층<br>B. 셀룰로스<br>C. 콜라겐<br>D. 글리코겐","A. 인지질 이중층 — 세포막의 기본 구조이며 단백질이 삽입되어 있다.","세포생물학 객관식"
"식물세포에만 세포벽이 있고 동물세포에는 없다. (O/X)","O — 식물세포는 셀룰로스 세포벽을 가지며, 동물세포에는 세포벽이 없다.","세포생물학 OX"
```

**Front 필드 구성 규칙:**
- 단답형: 질문 그대로
- 객관식: 질문 + `<br><br>` + 선택지를 `<br>`로 구분
- OX: 질문 + ` (O/X)`

**Back 필드 구성 규칙:**
- `정답 — 해설` 형식으로 작성
- Anki HTML 렌더링을 활용해 가독성 향상

**Tags 필드:** `강의제목 유형` (예: `세포생물학 단답형`)

```javascript
const downloadAnkiCSV = () => {
  const BOM = '\uFEFF';
  const headers = 'Front,Back,Tags\n';
  const tag = quizData.title.replace(/\s+/g, '');
  const rows = quizData.questions.map(q => {
    const typeTag = q.type === 'multiple_choice' ? '객관식' : q.type === 'short_answer' ? '단답형' : 'OX';
    let front = q.question;
    if (q.type === 'multiple_choice') {
      front += '<br><br>' + q.options.join('<br>');
    } else if (q.type === 'ox') {
      front += ' (O/X)';
    }
    const back = `${q.answer} — ${q.explanation}`;
    const tags = `${tag} ${typeTag}`;
    // 쌍따옴표 내부 따옴표 이스케이프
    const escapedFront = front.replace(/"/g, '""');
    const escapedBack = back.replace(/"/g, '""');
    return `"${escapedFront}","${escapedBack}","${tags}"`;
  }).join('\n');
  const blob = new Blob([BOM + headers + rows], { type: 'text/csv;charset=utf-8;' });
  const url = URL.createObjectURL(blob);
  const a = document.createElement('a');
  a.href = url;
  a.download = `${quizData.title}_Anki.csv`;
  a.click();
  URL.revokeObjectURL(url);
};
```

---

## Step 4: JSX 컴포넌트 템플릿

아래 구조를 기반으로 퀴즈 데이터를 채워 넣은 완성된 JSX를 생성한다.

```jsx
import { useState } from "react";

const quizData = { /* 생성된 JSON 데이터 */ };

export default function QuizApp() {
  const [currentIndex, setCurrentIndex] = useState(0);
  const [userAnswers, setUserAnswers] = useState({});
  const [showResult, setShowResult] = useState(false);
  const [showFeedback, setShowFeedback] = useState(false);
  const [inputValue, setInputValue] = useState("");

  const current = quizData.questions[currentIndex];
  const total = quizData.questions.length;
  const progress = ((currentIndex) / total) * 100;

  // 채점 함수 (단답형: 공백/대소문자 무시)
  const checkAnswer = (userAns, correctAns) => {
    return userAns.trim().toLowerCase().replace(/\s+/g, '') === 
           correctAns.trim().toLowerCase().replace(/\s+/g, '');
  };

  // Anki 호환 CSV 다운로드 함수
  const downloadAnkiCSV = () => {
    const BOM = '\uFEFF';
    const headers = 'Front,Back,Tags\n';
    const tag = quizData.title.replace(/\s+/g, '');
    const rows = quizData.questions.map(q => {
      const typeTag = q.type === 'multiple_choice' ? '객관식' : q.type === 'short_answer' ? '단답형' : 'OX';
      let front = q.question;
      if (q.type === 'multiple_choice') front += '<br><br>' + q.options.join('<br>');
      else if (q.type === 'ox') front += ' (O/X)';
      const back = `${q.answer} — ${q.explanation}`;
      return `"${front.replace(/"/g,'""')}","${back.replace(/"/g,'""')}","${tag} ${typeTag}"`;
    }).join('\n');
    const blob = new Blob([BOM + headers + rows], { type: 'text/csv;charset=utf-8;' });
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url; a.download = `${quizData.title}_Anki.csv`; a.click();
    URL.revokeObjectURL(url);
  };

  // 결과 화면, 문제 화면 등 렌더링...
  return (/* JSX UI */);
}
```

---

## 디자인 가이드라인

- 깔끔하고 학습에 집중할 수 있는 UI
- 색상: 정답 → 초록(#22c55e), 오답 → 빨강(#ef4444), 강조 → 남색(#3b82f6)
- 모바일 친화적 레이아웃
- Tailwind CSS 유틸리티 클래스 사용
- 문제 유형별 배지 색상: 객관식(파랑), 단답형(보라), OX(주황)

---

## 출력물

1. **JSX 아티팩트**: 완성된 퀴즈 앱 (claude.ai 내에서 바로 실행 가능), 30문제
2. **Anki CSV 다운로드**: 앱 내 버튼으로 제공 — Front/Back/Tags 형식으로 Anki에 바로 임포트 가능
3. **Anki 임포트 안내**: 다운로드 버튼 근처에 간단한 임포트 방법 표시

퀴즈 앱을 artifact로 생성하면 사용자가 바로 브라우저에서 풀 수 있다.

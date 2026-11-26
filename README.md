# 🪙🛡️ 돈킴이 (DonKimi)

> **AI 기반 지능형 피싱 탐지 플랫폼**
> GPT를 활용한 소프트웨어 개발 캡스톤 디자인 · Team 5 (2026년 1학기)

URL, 전화번호, 문자 캡처 이미지, 통화 녹음 파일을 입력하면 AI가 피싱 위험도를 진단하고,
진단 결과를 바탕으로 챗봇이 대응 방법을 안내하는 웹 서비스입니다.

> 이 저장소는 팀 프로젝트 원본 저장소 [Seonggyu0806/team5](https://github.com/Seonggyu0806/team5)를 개인 포트폴리오용으로 가져온 것입니다.

---

## 📌 주요 기능

| 기능 | 설명 |
| :--- | :--- |
| 🔗 URL 분석 | Google Safe Browsing과 AI로 악성 URL 위험도 판별 |
| 📱 전화번호 조회 | 신고된 피싱 번호 조회 및 직접 신고 |
| 🖼️ 이미지 검사 | 문자 캡처에서 OCR로 텍스트를 추출한 뒤 스미싱 분석 |
| 🎙️ 음성 검사 | 통화 녹음을 STT로 변환한 뒤 보이스피싱 패턴 분석 |
| 🤖 AI 챗봇 상담 | 진단 결과를 문맥으로 받아 맞춤형 대응 가이드 제공 |
| 👤 마이페이지 | 분석 이력과 대화 이력 조회 |

위험 등급은 5단계입니다: `SAFE`(0~20%) · `LOW`(20~40%) · `MEDIUM`(40~60%) · `HIGH`(60~80%) · `CRITICAL`(80~100%)

---

## 👥 팀 구성

| 파트 | 팀원 |
| :--- | :--- |
| Frontend | 박성규, 송용욱 |
| Backend | 최유진, **한태경** |
| AI / Data | 박성규, 송용욱, 최유진, 한태경 |

---

## 🛠 기술 스택

- **Backend:** Java 17, Spring Boot, Spring Security, JWT, Spring Data JPA, Swagger(springdoc)
- **Frontend:** React 19, TypeScript, Vite, Tailwind CSS, TanStack Query
- **Database / Infra:** MySQL (AWS RDS), AWS EC2, Vercel
- **AI / 외부 API:** OpenAI API (gpt-4o-mini), Naver CLOVA Speech (STT), Google Cloud Vision (OCR), Google Safe Browsing

---

## 🙋 내가 담당한 부분 (한태경 · Backend)

### 1. 음성·이미지 텍스트 추출 (STT / OCR)

**STT: `SttService`**
- Naver CLOVA Speech Recognition API로 통화 녹음(.mp3, .m4a)을 한국어 텍스트로 변환
- `RestTemplate`으로 파일을 `APPLICATION_OCTET_STREAM` 형식으로 전송하고, 언어를 한국어(`Kor`)로 지정해 인식률 향상
- 업로드 한도를 `max-file-size: 10MB`로 설정해 대용량 파일 업로드 시 서버 오류 방지

**OCR: `OcrService`**
- Google Cloud Vision API로 문자 메시지, 가짜 은행 앱 캡처 화면의 텍스트 추출
- `MultipartFile` 이미지를 Base64로 인코딩해 전송하고, 응답 JSON에서 `fullTextAnnotation`만 골라 정제된 텍스트로 반환
- 글자가 없는 이미지에 대한 방어 로직(`isEmpty()`)으로 서버 오류 방지

### 2. 하이브리드 AI 판별 엔진 설계

텍스트 추출과 문맥 판단을 분리한 멀티모달 분석 파이프라인을 구성했습니다.

```
음성 파일 ─▶ Naver CLOVA STT ─┐
                              ├─▶ OpenAI (문맥·위험도 분석) ─▶ 위험 등급 + 판단 근거
캡처 이미지 ─▶ Google Vision OCR ┘
```

| 서비스 | 역할 |
| :--- | :--- |
| `SttService` | 음성에서 텍스트 추출 |
| `OcrService` | 이미지에서 텍스트 추출 |
| `OpenAiService` | 프롬프트 기반 스미싱·보이스피싱 문맥 분석 |
| `VoiceAnalysisService` | STT와 OpenAI를 묶은 음성 분석 흐름 |
| `ImageAnalysisService` | OCR과 OpenAI를 묶은 이미지 분석 흐름 |

**설계 포인트:** 무거운 파일 분석을 OpenAI에 모두 맡기지 않았습니다. 텍스트 추출은 빠르고 저렴한 Naver/Google API에, 판단은 OpenAI에 나눠 맡겨서 처리 속도와 API 비용을 함께 줄였습니다.

### 3. URL 분석 흐름 재설계

- Google Safe Browsing이 **안전**으로 판단하면 AI를 호출하지 않고 바로 `SAFE`를 반환
- Google Safe Browsing이 **악성**으로 판단한 경우에만 AI가 2차 상세 분석
- 불필요한 OpenAI 호출을 줄여 비용을 절감하고, 안전한 사이트를 위험하다고 오판하는 일도 줄임

### 4. AI 모델 및 프롬프트 개선

- 모델을 `gpt-3.5-turbo`에서 `gpt-4o-mini`로 변경
- 응답 첫 줄에 `위험도: SAFE` 형식으로 반드시 출력하도록 형식을 통일해 파싱 안정성 확보
- 공인 도메인(google.com, naver.com, kakao.com 등)은 SAFE로 처리하는 원칙을 프롬프트에 명시
- 이미지(스미싱), 음성(보이스피싱), 채팅 상담용 프롬프트를 각각 분석 기준 5개 항목 중심으로 다시 작성

### 5. 백엔드 안정화 및 트러블슈팅

| 문제 | 원인 | 해결 |
| :--- | :--- | :--- |
| AI 챗봇 응답 저장 시 **500 오류** | `chat_message.message` 컬럼이 `VARCHAR(255)`라 긴 GPT 응답이 저장되지 않음 (`Data too long`) | 엔티티에 `@Column(columnDefinition = "TEXT")`를 적용하고, 운영 DB에 `ALTER TABLE`을 직접 실행. `ddl-auto: update`는 기존 컬럼 타입을 바꾸지 않는다는 점을 확인 |
| google.com도 "주의(75점)"로 **오판정** | `parseRiskLevel()`이 응답에 "위험"이라는 단어나 숫자 7, 8만 있어도 HIGH로 분류 ("위험하지 않습니다" → HIGH) | 첫 줄의 `위험도: XXX` 형식을 우선 파싱하고, 없을 때만 키워드 순서(`CRITICAL` → `HIGH` → `SAFE` → `LOW`)로 판정 |
| 이미 신고한 번호를 다시 조회하면 **400 오류** | 중복 신고를 `IllegalArgumentException`으로 처리 | 예외 대신 `alreadyReported: true`가 담긴 정상 응답을 반환해서 프론트엔드가 상태를 구분할 수 있게 변경 |

### 6. 협업

- 프론트엔드 PR(#5~#27)을 리뷰하고 병합(merge)
- 최종 백엔드 코드를 정리해 저장소에 반영
- 주차별 위키에 백엔드 작업 내역 문서화 (6주차, 7주차, 14주차)

---

## 📂 프로젝트 구조

```
├── backend/     # Spring Boot API 서버
│   └── src/main/java/com/phishing/
│       ├── config/       # Security, JWT, Swagger, 예외 처리
│       ├── controller/   # REST API
│       ├── domain/       # JPA 엔티티
│       ├── dto/
│       ├── repository/
│       └── service/      # STT, OCR, OpenAI, URL 분석 등 핵심 로직
└── frontend/    # React + TypeScript 웹 클라이언트
```

---

## 📖 문서

- 주차별 개발 기록: [Wiki](../../wiki)

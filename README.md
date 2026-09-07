# B2B WMS AI Platform — Frontend

중고·반품 도서의 입고 검수부터 출고까지를 다루는 물류센터 웹 애플리케이션입니다.
현장 작업자가 쓰는 모바일 화면과 관리자가 쓰는 데스크톱 화면이 한 앱에 들어 있습니다.

- **작업자(모바일)**: 도서 촬영, 바코드 스캔, 입고 적치, 출고 피킹
- **관리자(데스크톱)**: AI 검수 결과 확인, HITL 판정, 재고·주문 관리, 대시보드

백엔드: [wms-core-backend](https://github.com/newzed-enhancement/wms-core-backend)

---

## 기술 스택

| 영역 | 사용 |
|---|---|
| 프레임워크 | Next.js 16 (App Router), React 19, TypeScript |
| 스타일 | Tailwind CSS v4, shadcn/ui, Base UI, Lucide |
| 상태 | Jotai (전역 세션·업로드 큐), TanStack Query (서버 상태·캐싱) |
| 카메라·스캔 | WebRTC `getUserMedia` 커스텀 뷰파인더, ZXing (바코드) |
| 이미지 | browser-image-compression (Web Worker 압축), Canvas 전처리 |
| 3D | React Three Fiber (적재 시각화) |
| 테스트 | Vitest, Testing Library, MSW |

---

## 눈여겨볼 구현

### 촬영 직후 흔들림을 걸러냅니다

흔들린 사진을 서버로 보내면 AI 판독이 어긋나고 재촬영 왕복이 생깁니다. 그래서 업로드
전에 브라우저에서 먼저 거릅니다.

`src/features/inbound/utils/image-processor.ts`가 **Laplacian Variance**로 엣지 강도를
재서 흔들림 점수를 냅니다. 모바일 성능을 위해 이미지 전체가 아니라 **중앙 400×400 영역만**
잘라 연산합니다.

> OpenCV.js(WASM)를 쓰지 않고 순수 Canvas로 구현했습니다. WASM 런타임을 싣는 비용 대비
> 이 정도 연산에는 과했습니다.

### 촬영이 업로드를 기다리지 않습니다

작업자가 한 권 찍고 업로드가 끝날 때까지 서 있으면 현장 속도가 나오지 않습니다.
촬영 즉시 Jotai `uploadQueueAtom`에 넣고 다음 책으로 넘어가며, 압축과 S3 업로드는
백그라운드에서 진행됩니다.

이미지는 **백엔드를 거치지 않고 S3로 직접 올라갑니다**. 백엔드는 Presigned URL만 내주고
(인증 확인 후), 실제 바이트는 브라우저 → S3로 갑니다.

### 검수 진행 상황을 실시간으로 받습니다

AI 검수는 수십 초가 걸리는 비동기 작업이라 폴링으로 훑으면 서버에 부담이고 반응도 늦습니다.
`src/hooks/useJobStatus.ts`가 SSE로 상태를 받습니다.

- 1회용 티켓을 발급받아 연결합니다 (URL이 노출돼도 타 테넌트 스트림을 못 봅니다)
- 끊기면 **지수 백오프로 재연결**하고, 반복 실패하면 **폴링으로 자동 전환**합니다
- 토큰이 만료돼 401이 나면 갱신 후 티켓을 다시 받습니다

### HITL 대시보드의 동시 편집 방어

관리자 여러 명이 같은 검수 건을 동시에 판정하면 재고가 꼬입니다.

- 티켓을 잡는 순간 서버에 선점을 걸고 담당자를 표시합니다
- 승인·반려 버튼은 통신 중 잠급니다 (연타로 인한 중복 처리 방지)
- 낙관적 업데이트가 서버에서 실패하면 **화면을 원래대로 되돌립니다**

---

## 구조

```
src/
├── app/            # Next.js App Router (라우트별 페이지)
│   ├── worker/     #   작업자 모바일 화면 (입고·반품·출고)
│   ├── admin/      #   관리자 화면 (검수·재고·큐·발주)
│   └── api/        #   Route Handler (S3 Presigned URL 발급)
├── features/       # 기능 단위 묶음 (api · components · hooks · types)
│   ├── auth/  inbound/  inspections/  inventory/  lpn/  queue/
│   ├── orders/  outbound/  picking/  restock/  notifications/  dashboard/
├── components/     # 공용 UI, 레이아웃, 데이터 그리드
├── hooks/          # 공용 훅 (SSE 구독 등)
├── lib/            # axios 인스턴스, 토큰 갱신 인터셉터
└── mocks/          # MSW 핸들러 (백엔드 없이 개발할 때)
```

기능을 고칠 때 폴더 하나만 보면 되도록, `features/<기능>/` 안에 API 호출·컴포넌트·훅·타입을
함께 둡니다.

---

## 시작하기

```bash
npm ci
npm run dev        # http://localhost:3001
```

`.env.local`에 아래가 필요합니다. 실제 값은 팀 채널 공지를 참고하세요.

```env
NEXT_PUBLIC_API_URL=http://localhost:8080   # 백엔드 주소
NEXT_PUBLIC_DISABLE_MSW=true                # 실 백엔드 사용 (false면 MSW 목 사용)

# S3 Presigned URL 발급용 (서버 사이드에서만 사용)
OSS_REGION=ap-northeast-2
OSS_BUCKET_NAME=<버킷명>
OSS_ACCESS_KEY_ID=<팀 공지 참조>
OSS_ACCESS_KEY_SECRET=<팀 공지 참조>
CLOUDFRONT_DOMAIN=<CloudFront 도메인>
```

백엔드 없이 UI만 보려면 `NEXT_PUBLIC_DISABLE_MSW=false`로 두면 MSW가 API를 가로챕니다.

---

## 품질 게이트

PR을 올리면 아래가 **모두 통과해야** 머지할 수 있습니다.

```bash
npm run lint       # ESLint (error 0건이어야 통과, warning은 허용)
npm run test:run   # Vitest (47개 파일)
npm run build      # 프로덕션 빌드
```

기존 타입 부채(`any` 사용)와 React Compiler 경고는 `eslint.config.mjs`에서 **사유와 함께
warning으로 낮춰** 두었습니다. 숨긴 것이 아니라, error로 두면 CI가 상시 실패해서 새로
들어오는 위반을 못 걸러내기 때문입니다. 새로 `any`를 늘리지 말아 주세요.

---

## 문서

- [고도화 참여자 온보딩](https://github.com/newzed-enhancement/wms-core-backend/blob/main/docs/ONBOARDING.md) — **새로 합류했다면 먼저 읽으세요.** 백엔드 레포에 정본이 있습니다
- [프론트엔드 개발 가이드](FRONTEND_GUIDE.md) — 컴포넌트 작성 규칙, 상태 관리 패턴
- [기획서](docs/B2B_WMS_AI_Platform_기획서_ver1.4.2.0.md)
- [워크플로우](docs/B2B_WMS_AI_Platform_워크플로우_ver1.4.2.0.md)

## 만든 사람들

KT AIVLE School 9기 AI 2반 5조 빅프로젝트.

- **PM · 아키텍처**: 장문경
- **프론트엔드**: 박준희(API 연동·상태 관리·SSE), 고영빈(레이아웃·퍼블리싱·PWA), 소한민(모바일 현장 UI)
- **백엔드**: 박민우, 서다은 · **AI**: 홍경표

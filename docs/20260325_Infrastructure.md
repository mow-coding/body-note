[Initial Created: 2026-03-25]
[Last Modified: 2026-03-25 12:17]

# Body Note Infrastructure Guide

## 목적

Body Note의 Supabase, Vercel, 환경 변수, CLI 사용법을 팀원이 같은 방식으로 재현할 수 있도록 정리한 기준 문서입니다.

## 이번 결정 요약

- 앱 배포 단위는 `apps/web` 하나를 기준으로 둡니다.
- 브라우저용 Supabase 클라이언트는 `@supabase/supabase-js`로 초기화합니다.
- 루트의 공통 CLI는 `supabase`만 고정 설치하고, `vercel`은 보안과 감사 경고 최소화를 위해 `npx vercel` 방식으로 실행합니다.
- 민감 정보는 `.env`에만 두고, 저장소에는 `.env.example`만 남깁니다.
- 로컬 Supabase DB 실행은 Docker Desktop이 설치되어 있어야 가능합니다.

## 보안 원칙

절대 커밋하면 안 되는 항목

- `.env`, `.env.local`, `.env.*.local`
- Supabase Access Token
- Supabase DB 비밀번호
- 공개 저장소에 올리면 안 되는 내부 테스트 키와 임시 토큰
- Vercel 프로젝트 연결 정보와 실제 배포 환경 변수
- 로컬 백업 덤프, 사용자 데이터 export, 운영 로그

주의

- `VITE_`로 시작하는 환경 변수는 브라우저 번들에 포함될 수 있습니다.
- 따라서 브라우저에서 읽어야 하는 값만 `VITE_` 접두어를 사용합니다.
- `service_role` 같은 서버 전용 키는 절대 `VITE_` 접두어를 붙이면 안 됩니다.

## 현재 추가된 파일

- `.env`
- `.env.example`
- `vercel.json`
- `supabase/config.toml`
- `supabase/seed.sql`
- `apps/web/src/lib/supabase/client.ts`

## 환경 변수 목록

`.env.example` 기준

```env
VITE_SUPABASE_URL=
VITE_SUPABASE_PUBLISHABLE_KEY=
SUPABASE_ACCESS_TOKEN=
SUPABASE_PROJECT_ID=
SUPABASE_DB_PASSWORD=
VERCEL_ORG_ID=
VERCEL_PROJECT_ID=
```

설명

- `VITE_SUPABASE_URL`: 브라우저에서 사용하는 Supabase 프로젝트 URL
- `VITE_SUPABASE_PUBLISHABLE_KEY`: 브라우저에서 사용하는 publishable key
- `SUPABASE_ACCESS_TOKEN`: Supabase CLI 로그인 및 링크 작업용 토큰
- `SUPABASE_PROJECT_ID`: 원격 Supabase 프로젝트 ref
- `SUPABASE_DB_PASSWORD`: 원격 DB 연결 또는 링크 작업에 쓰는 비밀번호
- `VERCEL_ORG_ID`, `VERCEL_PROJECT_ID`: Vercel CLI 연결 자동화에 필요한 값

## 팀 공통 설치 명령어

처음 한 번만 실행

```bash
npm install
```

PowerShell에서 `npm`이 막히면

```powershell
npm.cmd install
npm.cmd run dev
```

## 루트 스크립트

```bash
npm run dev
npm run build
npm run lint
npm run supabase:init
npm run supabase:start
npm run supabase:stop
npm run supabase:status
npm run supabase:login
npm run supabase:db:reset
npm run vercel:dev
npm run vercel:link
```

## Supabase CLI 사용 순서

1. 의존성 설치

```bash
npm install
```

2. Supabase CLI 확인

```bash
npx supabase --version
```

3. 로컬 설정 초기화

```bash
npm run supabase:init
```

4. 계정 로그인

```bash
npm run supabase:login
```

5. 필요 시 원격 프로젝트 연결

```bash
npx supabase link --project-ref <your-project-id>
```

6. Docker Desktop 실행 후 로컬 DB 시작

```bash
npm run supabase:start
```

7. 상태 확인

```bash
npm run supabase:status
```

8. 로컬 DB 종료

```bash
npm run supabase:stop
```

## Docker 관련 주의사항

현재 이 개발 머신에서는 `docker` 명령이 없어 로컬 Supabase DB까지는 아직 실행하지 못했습니다.

즉, 지금 완료된 범위는 아래와 같습니다.

- Supabase CLI 설치 완료
- `supabase init` 완료
- `supabase/config.toml` 생성 완료
- Vite 기본 포트 `5173`에 맞춘 auth redirect URL 보정 완료

로컬 DB 실행 전 필수 조건

- Docker Desktop 설치
- Docker Desktop 실행 상태 확인

## Supabase 클라이언트 초기 설정

브라우저용 클라이언트는 아래 파일에서 초기화합니다.

- `apps/web/src/lib/supabase/client.ts`

정책

- 환경 변수가 없으면 즉시 에러를 던져 잘못된 배포를 빨리 발견합니다.
- 싱글턴 클라이언트로 중복 초기화를 막습니다.

## Vercel 설정

루트 `vercel.json` 기준

- `framework`: `vite`
- `installCommand`: `npm install`
- `buildCommand`: `npm run build`
- `devCommand`: `npm run dev`
- `outputDirectory`: `apps/web/dist`

추가 보안 헤더

- `X-Content-Type-Options: nosniff`
- `Referrer-Policy: strict-origin-when-cross-origin`
- `X-Frame-Options: DENY`

Vercel CLI 운영 방식

- 저장소 의존성으로 고정하지 않고 `npx vercel ...`로 실행합니다.
- 이유는 현재 버전의 Vercel CLI가 다수의 dev-only audit 경고를 끌고 오기 때문입니다.
- 공개 저장소 기준에서는 불필요한 의존성을 줄이는 쪽이 더 안전합니다.

## 검증 결과

- `npm run build` 통과
- `npm run lint` 통과
- `npx supabase --version` 확인 완료
- `npm audit --omit=dev` 결과 `0 vulnerabilities`
- `docker --version`은 실패했고, 따라서 `npm run supabase:start`는 Docker 설치 전까지 보류입니다.

## 연결 완료 상태

- GitHub 공개 저장소 `mow-coding/body-note` 생성 및 `main` 푸시 완료
- Supabase CLI 로그인 완료
- Supabase `BodyNote` 프로젝트 ref `stgtezhzaopensronbag` 링크 완료
- Vercel `Blue` 팀 아래 `body-note` 프로젝트 생성 및 로컬 링크 완료
- Production 배포 완료: `https://body-note.vercel.app`

주의

- Vercel의 `Production`과 `Development` 환경 변수는 설정 완료했습니다.
- `Preview` 환경 변수는 Vercel CLI가 브랜치 선택을 추가로 요구해 두 값이 아직 비어 있습니다.
- 현재 앱은 이 값이 즉시 필요하지 않아 배포는 성공했지만, 추후 PR 미리보기 배포에서 Supabase 값을 직접 사용하기 시작하면 Vercel 대시보드에서 같은 두 값을 `Preview`에도 추가하는 것이 안전합니다.

## 팀 온보딩 체크리스트

1. 저장소 clone
2. 루트에서 `npm install`
3. `.env.example`를 보고 `.env` 채우기
4. PowerShell이면 `npm.cmd run dev` 사용
5. Supabase 작업이 필요하면 `npm run supabase:login`
6. 로컬 DB가 필요하면 Docker Desktop 실행 후 `npm run supabase:start`

## 공식 참고 문서

- [Supabase CLI Local Development](https://supabase.com/docs/guides/local-development/cli/getting-started)
- [Supabase CLI Reference](https://supabase.com/docs/reference/cli/introduction)
- [Supabase JavaScript Client](https://supabase.com/docs/reference/javascript/installing)
- [Vercel Project Configuration](https://vercel.com/docs/project-configuration)

## Next Review Session

현재 시스템은 GitHub, Supabase, Vercel 연결과 Production 배포까지 완료된 상태이며, 다음 리뷰에서는 Docker Desktop 설치 후 로컬 Supabase DB와 Preview 환경 변수만 이어서 점검하면 됩니다.

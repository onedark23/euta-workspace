---
title: ADR - 043 Microsoft 로그인 구현
path:  euta-workspace / 40.설계 / ADR - 043 Microsoft-로그인-구현
ref: PLAN 결정 "로그인"(2026-10-30) · SPEC_클라우드-계정-프로젝트 "가입·로그인" · ARCH_euta-cloud · ADR-023(클라우드 인증) · PLAN_euta-cloud C10
---

# ADR - 043 Microsoft 로그인 구현

## 상태

채택 (2026-11-02, 단계 C10). 코드: `euta-cloud` (`packages/server/src/auth.ts` · `config.ts`, `packages/web/src/pages/auth.tsx`, 배포 예시 `k8s/example`). 조직의 Entra 앱 `euta-cloud-login` 등록 후 dev 배포(서버 `4344e37`)에서 **회사 계정 실제 로그인 확인**(사용자, 2026-11-02).

## 맥락

- PLAN 결정 "로그인"(2026-10-30): 회사 Microsoft 계정(Entra ID, 회사 테넌트 한정)이 주 입구, 첫 로그인 = 가입(바로 활성), 이메일 새 가입은 닫고 기존 이메일 회원 로그인만 남긴다. 사내 도구에서 흔한 배치.
- 인증 계층은 Better Auth(ADR-023). Better Auth 에 Microsoft 제공자가 있다.
- 정하지 않은 세부: 사용자 정보를 어디서 읽을지(Graph 권한 여부), 같은 메일 기존 계정 연결 규칙, 미인증 계정의 비밀번호, 실패 사유를 웹에 전하는 방법, 비밀 값 배치.

## 결정

### 설정

- `MICROSOFT_TENANT_ID` · `MICROSOFT_CLIENT_ID` · `MICROSOFT_CLIENT_SECRET` — **셋 다 있거나 셋 다 없어야** 한다. 일부만 있으면 서버가 시작을 거절한다(조용히 꺼진 채로 뜨는 것보다 낫다). 테넌트가 `common` · `organizations` · `consumers` 면 거절 — 단일 테넌트만.
- 셋 다 없으면 이전 동작(이메일 가입 + SMTP 인증 또는 관리자 승인). 공개 설정 `GET /api/config` 에 `microsoftLogin` 을 더해 웹이 화면을 고른다.
- 앱 등록: 조직의 **Entra 앱 등록**(단일 조직 앱) — 웹 앱(서버), 위임 권한 `openid profile email` 만, 애플리케이션 권한 없음. 비밀값 12개월 만료 → 조직 절차로 재발급 후 `create-secrets.sh --microsoft` + 재시작(절차: `euta-cloud/k8s/example/README.md`).
- dev 배치: 테넌트 ID · 클라이언트 ID 는 ConfigMap(비밀 아님), 클라이언트 비밀은 Secret `euta-cloud-secrets` 의 `MICROSOFT_CLIENT_SECRET`(서버 env 는 `optional: true`). `create-secrets.sh --microsoft` 가 가려진 입력으로 받아 `--patch-file` 로 넣는다 — 명령 인자 · 화면 · 저장소에 값이 남지 않는다. `--rotate` 는 이 키를 유지한다.

### 제공자

- Better Auth `socialProviders.microsoft` + `tenantId` → authority 가 회사 테넌트(`login.microsoftonline.com/<tenant>/…`).
- 범위 `openid profile email` 만(`disableDefaultScope`) — Graph 권한(`User.Read`)이 없어 관리자 동의가 필요 없다. 사진은 받지 않는다.
- `prompt=select_account` — 여러 계정을 쓰는 사람이 고를 수 있게.
- **사용자 정보 = id token 만**(`getUserInfo` 덮어씀, Graph 호출 없음). 토큰은 클라이언트 비밀로 회사 테넌트 토큰 엔드포인트에서 TLS 로 직접 받은 것이라 서명 검증은 생략한다(OIDC Core 3.1.3.7 ⑥). 그래도 `tid` 가 회사 테넌트가 아니면 거절한다.
- 회사 계정에 선택 클레임 `email` 이 없을 수 있어 **없으면 UPN(`preferred_username`)** 을 메일로 쓴다. 테넌트가 관리하는 주소이므로 `emailVerified: true`.
- 허용 도메인 검사는 기존 사용자 생성 훅이 그대로 한다(Microsoft 첫 로그인도 같은 훅).

### 이메일 가입 닫기

- `emailAndPassword.disableSignUp = (Microsoft 켜짐)` — API 에서 거절. 이메일·비밀번호 로그인은 그대로(기존 회원). 웹은 가입 링크를 숨기고 `/signup` 을 `/login` 으로 돌린다.

### 같은 메일 기존 계정 연결

- `accountLinking: { enabled, trustedProviders: ['microsoft'], requireLocalEmailVerified: false }` — 대소문자 무시 같은 메일이면 그 계정에 연결(프로젝트 · 역할 · 관리자 · 토큰 유지). 미인증 로컬 계정도 연결한다(아래 비밀번호 규칙 때문에 안전).
- **연결 훅은 `account.create.before`** 다. Better Auth 는 `create.after` 훅을 요청이 끝날 때(`runWithAdapter` 의 대기 목록)로 미룬다 — 즉 세션 생성 훅의 "승인 대기 거절" 검사 **뒤**에 돈다. 처음 `after` 로 짰을 때 승인 대기 계정이 활성화되기 전에 세션이 거절됐다. `before` 는 연결 행을 쓰기 직전에 바로 돈다. 새 Microsoft 사용자는 같은 순간 아직 커밋 전이라 훅이 찾지 못하고 넘어간다(그 사용자는 사용자 생성 훅이 이미 활성으로 만든다).
- 훅이 하는 일:
  1. **메일 미인증이면서 관리자가 아닌 계정**: 비밀번호(`credential` 계정 행)를 지우고 `emailVerified = true`. 메일함을 증명하지 않은 비밀번호는 남의 메일로 먼저 가입한 사람이 정했을 수 있다 — 연결 뒤 메일이 인증됨으로 바뀌면 그 비밀번호가 그대로 열리게 된다. 승인 모드의 승인 대기 계정, 메일 모드에서 인증 메일을 열지 않은 계정이 여기에 든다.
  2. 승인 대기 → 활성. 비활성은 그대로(세션 생성 훅이 `ACCOUNT_DEACTIVATED` 로 거절).
- **결정 출처**: 사용자에게 "미인증이면 모두 삭제 / 승인 대기만 삭제"를 물었고 **"승인 대기만"**(관리자 비밀번호 유지)을 골랐다(2026-11-02). 질문에 없던 **메일 인증 모드의 미인증 계정**(상태는 활성)은 구현 판단으로 삭제 대상에 넣었다 — Better Auth 가 연결 때 `emailVerified` 를 참으로 바꾸므로, 빼면 남이 정한 비밀번호가 그때부터 열린다. 지금 dev 는 승인 모드(SMTP 없음)라 결과는 "승인 대기만"과 같다. 글자 그대로 승인 대기만으로 좁히려면 조건을 `status = 'pending'` 으로 바꾸면 된다.
- **관리자 예외(사용자 결정 2026-11-02)**: 승인 모드에서 `ADMIN_EMAILS` 로 바로 활성 가입한 관리자는 메일 미인증이지만 비밀번호를 남긴다. 대안 "미인증이면 모두 삭제"는 Microsoft 를 끄고 SMTP 도 없을 때 관리자가 비밀번호를 재설정할 길이 없어 잠길 수 있어 택하지 않았다. 누가 관리자 메일로 먼저 가입했다면 그 비밀번호가 남지만, 그 사람은 이미 관리자 권한을 가진 상태라 Microsoft 연결이 권한을 넓히지는 않는다.
- 관리자가 승인한 계정은 승인 때 `emailVerified = true` 가 되므로 비밀번호가 남는다(SPEC "연결 뒤에도 비밀번호 로그인은 남는다").

### 실패 사유 → 웹

- 웹은 `POST /api/auth/sign-in/social`(`callbackURL = next`, `errorCallbackURL = /login?next=…`, `disableRedirect`)로 주소를 받아 이동한다. 장치 승인(`/device?user_code=…`)도 `next` 로 돌아온다.
- 실패하면 Better Auth 가 `errorCallbackURL?error=…` 로 돌려보내는데 **값의 모양이 제각각**이다: 우리 훅의 코드(`ACCOUNT_DEACTIVATED`, `PENDING_APPROVAL`), Better Auth 자체 코드(`unable_to_get_user_info` = 테넌트 다름 · `state_mismatch` 등), 사용자 생성 실패는 **메시지를 밑줄로 바꾼 것**(`sign-up_is_limited_to_…` = 허용 도메인 밖), Microsoft 취소는 `access_denied`. 웹은 이를 네 문구(비활성 · 회사 계정 아님 · 취소 · 그 밖)로 묶고, `PENDING_APPROVAL` 이면 승인 대기 화면으로 보낸다.

## 대안

- **Graph `/me` 로 사용자 정보**: `User.Read` 위임 권한과 동의가 필요하고 호출이 하나 늘어난다. id token 으로 충분해 택하지 않음.
- **연결을 `after` 훅 + 세션 검사 완화**: 세션 훅에서 Microsoft 로그인일 때만 대기 상태를 통과시키는 방식 — 세션 훅이 어떤 로그인인지 알 길이 깔끔하지 않고, 활성화가 요청 끝까지 미뤄져 실패 시 상태가 엇갈린다. `before` 가 단순하다.
- **연결 거부(같은 메일이면 오류)**: 기존 회원이 프로젝트를 잃거나 관리자가 계정을 합쳐 줘야 한다. PLAN 결정이 "연결"이다.

## 검증

- 서버(가짜 토큰 엔드포인트, `test/microsoft-login.test.ts`): 회사 테넌트 authority · 범위 · 공개 설정 · 이메일 가입 닫힘 · 새 사용자 바로 활성 + `callbackURL` 복귀 · UPN 대체 · 같은 메일 연결(비밀번호 유지) · 관리자 유지(비밀번호 유지) · 승인 대기 활성 + 비밀번호 삭제 · 비활성 거절 · 허용 도메인 밖 거절 · 다른 테넌트 거절 · 장치 승인 복귀 · 로그에 비밀 없음 · 메일 모드 미인증 계정(남이 만든 비밀번호 무효). 서버 전체 336개 통과.
- 웹 E2E(`e2e/microsoft-login.spec.ts`, `/api/config` 를 Microsoft 켜짐으로 바꿔 응답): 주 버튼 · 안내 · 접힌 이메일 로그인 · 가입 링크 없음 · `/signup` → `/login` · 시작 요청의 `callbackURL`/`errorCallbackURL` · 실패 사유 다섯 가지 · 캡처 3장. 웹 E2E 전체 22개 통과.
- dev 실제 로그인: 공개 설정 `microsoftLogin: true`, 로그인 시작 주소를 Microsoft 가 오류 없이 받음, 회사 계정 로그인 성공(사용자 확인 2026-11-02).

# Admin 보안 가이드 (공통)

> Next.js 기반 프로젝트의 어드민 페이지 구현 시 공통 적용 기준.  
> 프로젝트마다 복사하거나 참조해서 사용.

---

## 1. 인증 구조

- **비밀번호 보관** — `ADMIN_PASSWORD` 환경변수에만 존재 (서버 전용, `NEXT_PUBLIC_` 절대 사용 금지)
- **검증** — `POST /api/admin/auth`에서 서버사이드 비교 (`crypto.timingSafeEqual`로 타이밍 공격 방지)
- **세션** — HMAC-SHA256 서명된 토큰을 `httpOnly` 쿠키로 발급 (JS 접근 불가, 4시간 만료)
- **클라이언트** — 비밀번호를 저장하지 않고, fetch 후 즉시 `setPassword("")`로 초기화
- **전송** — 비밀번호는 POST body에 1회만 전송, 이후 모든 요청은 쿠키 기반 인증

---

## 2. 환경변수 보안

- `ADMIN_PASSWORD` — 최소 16자 이상 랜덤 문자열 사용
- `ADMIN_SECRET` — HMAC 서명용 시크릿 키 (다른 시크릿과 별도로 생성)
- 두 값 모두 `.env.local`에만 존재, `.env.example`에는 키 이름만 명시 (값 없이)
- `.gitignore`에 `.env.local` 반드시 포함 확인

```bash
# 랜덤 시크릿 생성
openssl rand -base64 32
```

---

## 3. API Route 보호

- 모든 `/api/admin/*` 라우트는 요청마다 쿠키 토큰 검증
- 토큰 검증 실패 시 `401 Unauthorized` 반환, 에러 메시지에 상세 정보 노출 금지
- `/admin` 페이지 접근 시 미인증이면 `/admin/login`으로 리다이렉트 (미들웨어 처리)

---

## 4. 브루트포스 방어

- 로그인 실패 5회 이상 시 IP 기반 15분 잠금
- 잠금 상태는 서버 메모리 또는 DB에 저장
- 실패 응답은 성공 응답과 동일한 시간 후 반환 (타이밍으로 계정 존재 여부 유추 방지)

---

## 5. 쿠키 설정

```ts
{
  httpOnly: true,       // JS 접근 차단
  secure: true,         // HTTPS에서만 전송
  sameSite: 'strict',   // CSRF 방지
  maxAge: 60 * 60 * 4,  // 4시간
  path: '/admin',       // admin 경로에서만 전송
}
```

---

## 6. CSRF 방어

- 세션 쿠키의 `sameSite: strict` 설정으로 기본 방어
- 추가로 요청 헤더에 `X-Requested-With: XMLHttpRequest` 확인

---

## 7. 파일 업로드 보안

- 허용 확장자: `.jpg` `.jpeg` `.png` `.webp` 만 허용
- MIME 타입 서버사이드에서 재검증 (클라이언트 값 신뢰 금지)
- 파일 크기 제한: 최대 5MB
- 업로드 파일명 UUID로 랜덤 재생성 (원본 파일명 그대로 사용 금지)
- Storage 버킷은 `public` 읽기 / `authenticated` 쓰기로 설정

---

## 8. Vercel 배포 시 주의사항

- 모든 환경변수는 Vercel 대시보드 → Settings → Environment Variables에서만 관리
- Preview 배포에는 `ADMIN_PASSWORD` 다른 값으로 분리 설정 권장
- 로그에 민감 정보 출력되지 않도록 `console.log`에 비밀번호/토큰 포함 금지

---

## 9. 모니터링

- 로그인 성공/실패 이벤트 DB `admin_logs` 테이블에 기록 (IP, 시각, 결과)
- 비정상 접근 패턴 감지 시 알림 (Slack webhook 연동 권장)

---

## 10. 정기 점검 체크리스트

- [ ] `ADMIN_PASSWORD` 3개월마다 교체
- [ ] 사용하지 않는 API Route 제거
- [ ] DB RLS(Row Level Security) 정책 주기적 검토
- [ ] npm 패키지 취약점 점검: `npm audit`

---

## 11. 클코(Claude Code) 사용 시

이 문서를 프로젝트 루트에 `SECURITY.md`로 저장 후 클코에게 전달:

```
SECURITY.md 읽고 이 보안 구조대로 /admin 페이지 구현해줘.
```

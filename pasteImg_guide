# 이미지 붙여넣기 업로드 — 공통 에러 가이드

> 클립보드 붙여넣기(paste) 방식으로 이미지를 업로드할 때 자주 발생하는 4가지 패턴.  
> 새 프로젝트에 붙여넣기 업로드 기능 구현 시 사전에 참고.

---

## 에러 1. Stale Closure — 이미지 중복/덮어쓰기

### 증상
붙여넣기로 이미지를 빠르게 여러 장 올리면, 먼저 업로드한 이미지가 사라지거나 덮어씌워짐.

### 원인
`uploadImage` 함수가 렌더링 시점의 옛 state를 클로저로 참조함.  
동시 업로드 시 나중에 완료된 업로드가 이전 결과를 모르고 덮어씀.

```ts
// ❌ 잘못된 패턴 — 렌더링 시점의 state를 직접 참조
const uploadImage = async (file) => {
  const url = await upload(file)
  setImages([...images, url]) // images가 stale할 수 있음
}
```

### 해결
`setState`의 함수형 업데이트로 항상 최신 state를 읽도록 수정.

```ts
// ✅ 올바른 패턴 — prev로 최신 state 보장
const uploadImage = async (file) => {
  const url = await upload(file)
  setImages(prev => [...prev, url])
}
```

---

## 에러 2. CDN 캐시 — 삭제한 이미지가 재등장

### 증상
이미지를 삭제하고 새 이미지를 올렸는데, 화면에 이전 이미지가 계속 보임.  
새로고침해도 동일.

### 원인
파일명이 고정(슬롯 번호, ID 기반)이라 같은 경로에 덮어쓰기 발생.  
Vercel Blob, S3, Cloudflare 등 CDN이 이전 URL을 캐시한 채로 반환.

```ts
// ❌ 잘못된 패턴 — 고정 파일명
const filename = `${projectId}_${slotIndex}.png`
```

### 해결
파일명에 타임스탬프 또는 UUID를 추가해 매 업로드마다 고유한 URL 생성.

```ts
// ✅ 올바른 패턴 — 고유 파일명
const filename = `${projectId}_${Date.now()}.png`
// 또는
const filename = `${projectId}_${crypto.randomUUID()}.png`
```

---

## 에러 3. image / images 필드 불일치 — 페이지마다 다른 이미지 표시

### 증상
어드민에서 이미지를 수정했는데, 메인 페이지에서는 이전 이미지가 보임.  
또는 어드민과 메인이 서로 다른 이미지를 표시.

### 원인
DB 또는 state에 `image` (대표 이미지 단수)와 `images` (배열) 필드가 분리되어 있는데,  
`images`만 업데이트하고 `image`를 동기화하지 않음.

```ts
// ❌ 잘못된 패턴 — images만 업데이트
setProject(prev => ({
  ...prev,
  images: newImages
}))
```

### 해결
`images` 수정 시 항상 `image = images[0]`으로 동기화.

```ts
// ✅ 올바른 패턴 — 항상 함께 동기화
setProject(prev => ({
  ...prev,
  images: newImages,
  image: newImages[0] ?? null
}))

// DB 업데이트 시도 동일하게 적용
await updateProject({
  images: newImages,
  image: newImages[0] ?? null
})
```

---

## 에러 4. DOM 포커스 의존 — 붙여넣기가 엉뚱한 슬롯에 들어감

### 증상
새 이미지 슬롯을 추가하고 붙여넣기 했는데, 이전 슬롯에 이미지가 들어감.  
슬롯을 클릭해도 포커스가 의도한 곳에 안 잡힘.

### 원인
각 `ImageSlot`에 `onPaste + tabIndex={0}`을 달아 DOM 포커스에 의존하는 구조.  
새 슬롯 추가 후에도 포커스가 이전 슬롯에 남아 있어서 붙여넣기 대상이 틀어짐.

```tsx
// ❌ 잘못된 패턴 — DOM 포커스에 의존
<div
  tabIndex={0}
  onPaste={(e) => handlePaste(e, slotIndex)}
>
```

### 해결
`pasteTarget` state로 붙여넣기 대상 슬롯을 명시적으로 추적하고,  
`document` 레벨 paste 핸들러에서 `pasteTarget`을 기준으로 이미지 전달.

**핵심 구조 4가지:**

1. **pasteTarget state** — 현재 붙여넣기 대상 슬롯을 명시적으로 추적
```ts
const [pasteTarget, setPasteTarget] = useState<{ projectId: string; slot: number } | null>(null)
```

2. **document 레벨 paste 핸들러** — DOM 포커스와 무관하게 동작
```ts
useEffect(() => {
  const handlePaste = (e: ClipboardEvent) => {
    if (!pasteTarget) return
    const file = Array.from(e.clipboardData?.items ?? [])
      .find(item => item.type.startsWith('image/'))
      ?.getAsFile()
    if (file) uploadToSlot(file, pasteTarget)
  }
  document.addEventListener('paste', handlePaste)
  return () => document.removeEventListener('paste', handlePaste)
}, [pasteTarget])
```

3. **클릭으로 슬롯 선택**
```tsx
<div onClick={() => setPasteTarget({ projectId, slot: index })}>
```

4. **새 슬롯 추가 시 자동 선택**
```ts
const addImageSlot = () => {
  const newSlot = images.length
  setImages(prev => [...prev, null])
  setPasteTarget({ projectId, slot: newSlot }) // 자동으로 새 슬롯 선택
}
```

5. **시각적 피드백** — 선택된 슬롯 표시
```tsx
const isTarget = pasteTarget?.projectId === projectId && pasteTarget?.slot === index

<div className={isTarget ? 'ring-2 ring-green-500' : ''}>
  {isTarget && <span>← 붙여넣기 대상</span>}
</div>
```

---

## 체크리스트 (구현 전 확인)

- [ ] `setState` 내부에서 이전 state를 참조할 때 함수형 업데이트(`prev =>`) 사용
- [ ] 업로드 파일명에 `Date.now()` 또는 `randomUUID()` 포함
- [ ] 단수/복수 이미지 필드(`image` / `images`) 구조 통일 또는 항상 동기화
- [ ] 이미지 삭제 시 CDN에 올라간 파일도 함께 삭제 처리 (orphan 방지)
- [ ] 붙여넣기 대상을 DOM 포커스가 아닌 `pasteTarget` state로 명시적 관리
- [ ] 새 슬롯 추가 시 자동으로 `pasteTarget` 업데이트
- [ ] 선택된 슬롯에 시각적 피드백 표시

---

## 클코(Claude Code) 사용 시

```
PASTE-IMAGE-UPLOAD.md 읽고, 이 가이드의 4가지 패턴을 피해서
붙여넣기 이미지 업로드 기능 구현해줘.
```

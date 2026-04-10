## CSS 전역 규칙

### 커서 깜빡임 방지
모든 웹 프로젝트 세팅 시 globals.css (또는 base CSS)에 반드시 포함:

```css
*:not(input):not(textarea):not([contenteditable]) {
  caret-color: transparent;
}
```

Tailwind 프로젝트면 `@layer base` 안에 넣을 것.
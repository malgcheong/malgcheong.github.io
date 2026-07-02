# 맑을 청 淸

백엔드 개발자의 AI 활용 기록 — [malgcheong.github.io](https://malgcheong.github.io/)

Java·Spring 백엔드 개발자가 최신 AI 모델 동향, 로컬 LLM 구성(Ollama 등), OpenAI·OpenRouter 같은 LLM API 활용기를 기록하는 블로그입니다. AI 요약 재탕 없이, 직접 돌려본 것만 씁니다.

## 기술 스택

- [Astro 5](https://astro.build/) + [Fuwari](https://github.com/saicaca/fuwari) 테마
- Tailwind CSS, Svelte, Pagefind(검색)
- GitHub Actions → GitHub Pages 자동 배포 (`main` push 시)

## 로컬 실행

```bash
pnpm install       # 의존성 설치 (Node 22, pnpm 9)
pnpm dev           # 개발 서버 → http://localhost:4321
pnpm build         # 프로덕션 빌드 (dist/)
pnpm preview       # 빌드 결과 미리보기
pnpm new-post -- 파일명   # 새 글 스캐폴드 생성
```

글은 `src/content/posts/`에 마크다운으로 작성합니다.

## 배너

홈 배너는 `public/banners/`의 이미지를 새로고침마다 랜덤으로 보여줍니다.
추가하려면 1920×800 이미지를 넣고 `src/banners.ts`의 `BANNERS` 배열에 경로를 추가하면 됩니다.

## 라이선스

- **코드**(Fuwari 테마 기반): [MIT](./LICENSE)
- **글·이미지 콘텐츠**: [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/)

# Next.js 개발자를 위한 Vitest & React Testing Library 사용 가이드

## 📌 1. Vitest 설정 심화

### Vitest 상세 환경설정 (`vite.config.ts`)
Vitest가 Next.js와 원활히 통합되도록 설정합니다.

```typescript
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    environment: 'jsdom', // 브라우저 환경을 시뮬레이션하기 위한 설정
    globals: true, // describe, it 등의 전역 함수 사용 가능
    setupFiles: ['./test/setupTests.ts'], // 테스트 전 초기화 스크립트
    coverage: {
      reporter: ['text', 'json', 'html'], // 커버리지 리포트 형식
      exclude: ['node_modules/', 'test/'], // 커버리지 제외 대상
    },
    alias: [
      { find: /^@\/(.*)/, replacement: '/src/$1' }, // @/ 경로 매핑
    ],
  },
});
```

### 글로벌 설정 파일 작성 (`test/setupTests.ts`)

```typescript
import '@testing-library/jest-dom'; // 사용자 정의 matcher를 위한 설정 (예: toBeInTheDocument)
import { cleanup } from '@testing-library/react';
import { afterEach } from 'vitest';

afterEach(() => cleanup()); // 각 테스트 후 DOM cleanup
```

## 📌 2. 실제 Vitest 적용 예시

### ① Next.js 페이지 테스트
```tsx
import { render, screen } from '@testing-library/react';
import { describe, it, expect } from 'vitest';
import HomePage from '@/pages/index';

describe('<HomePage />', () => {
  it('홈페이지 렌더링 테스트', () => {
    render(<HomePage />);
    expect(screen.getByRole('heading', { name: '홈페이지' })).toBeInTheDocument();
  });
});
```

### ② Next.js 컴포넌트 테스트
```tsx
import { render, screen, fireEvent } from '@testing-library/react';
import { it, expect, vi } from 'vitest';
import Button from '@/components/Button';

it('Button 클릭 이벤트 테스트', () => {
  const handleClick = vi.fn();
  render(<Button onClick={handleClick}>클릭</Button>);
  fireEvent.click(screen.getByText('클릭'));
  expect(handleClick).toHaveBeenCalled();
});
```

### ③ Next.js API 라우트 테스트
```typescript
import { it, expect, vi } from 'vitest';
import handler from '@/pages/api/user';

it('API 라우트 응답 테스트', async () => {
  const req = { method: 'GET' };
  const res = { status: vi.fn().mockReturnThis(), json: vi.fn() };
  await handler(req, res);
  expect(res.status).toHaveBeenCalledWith(200);
  expect(res.json).toHaveBeenCalledWith({ user: { id: 1, name: 'John' } });
});
```

### ④ Next.js 미들웨어 테스트
```typescript
import { NextRequest } from 'next/server';
import { middleware } from '@/middleware';
import { it, expect } from 'vitest';

it('미들웨어 정상 동작 테스트', () => {
  const request = new NextRequest('http://localhost/about');
  const response = middleware(request);
  expect(response.status).toBe(200);
});
```

### ⑤ Next.js Router 테스트
```typescript
import { renderHook } from '@testing-library/react';
import { useRouter } from 'next/router';
import { useCustomRouter } from '@/hooks/useCustomRouter';
import { it, expect, vi } from 'vitest';

vi.mock('next/router', () => ({
  useRouter: () => ({ push: vi.fn() }),
}));

it('라우터 hook 동작 테스트', () => {
  const { result } = renderHook(() => useCustomRouter());
  result.current.navigate('/about');
  expect(useRouter().push).toHaveBeenCalledWith('/about');
});
```

## 📌 3. React Testing Library 비동기 예제
```tsx
import { render, screen, waitFor } from '@testing-library/react';
import { it, expect } from 'vitest';
import UserProfile from '@/components/UserProfile';

it('비동기 데이터 렌더링 테스트', async () => {
  render(<UserProfile userId="1" />);
  await waitFor(() => expect(screen.getByText('John')).toBeInTheDocument());
});
```

## 📌 4. Vitest 성능 최적화 전략
```typescript
export default defineConfig({
  test: {
    threads: true, // 병렬 실행 활성화
    maxThreads: 4, // 최대 병렬 스레드 수
  },
});
```

```sh
vitest run ./test/components --coverage // 특정 디렉토리만 테스트 실행 및 커버리지 측정
```

## 📌 5. 테스트 커버리지 관리
```sh
vitest run --coverage --coverageThreshold='{"branches":80,"functions":85,"lines":90,"statements":90}'
```

## 📌 6. 베스트 프랙티스 상세 가이드

### 6.1 비동기 처리

비동기 테스트에서는 `waitFor`, `findBy`, `async/await` 등을 활용해 렌더링 완료 이후의 결과를 검증해야 합니다.

```tsx
import { render, screen, waitFor } from '@testing-library/react';
import { it, expect } from 'vitest';
import UserProfile from '@/components/UserProfile';

it('비동기 데이터 렌더링 테스트', async () => {
  render(<UserProfile userId="1" />);
  await waitFor(() => expect(screen.getByText('John')).toBeInTheDocument());
});
```

### 6.2 API 모킹에 MSW 적극 활용

Mock Service Worker(MSW)는 네트워크 요청을 가로채고 가짜 응답을 반환해 테스트의 신뢰성과 속도를 향상시킵니다.

```ts
// mocks/handlers.ts
import { rest } from 'msw';

export const handlers = [
  rest.get('/api/user', (req, res, ctx) => {
    return res(
      ctx.status(200),
      ctx.json({ id: 1, name: 'John Doe' }),
    );
  }),
];
```

```ts
// test/setupTests.ts
import { setupServer } from 'msw/node';
import { handlers } from '../mocks/handlers';

const server = setupServer(...handlers);

beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());
```

### 6.3 모듈별 명확한 테스트 시나리오 유지

모든 기능을 통합 테스트로 하지 말고 컴포넌트, 페이지, 훅, API 별로 테스트 파일을 나누고 명확한 책임을 부여하세요.

```sh
src/
├── components/
│   └── Button.test.tsx
├── pages/
│   └── HomePage.test.tsx
├── hooks/
│   └── useCustomRouter.test.ts
└── api/
    └── user.test.ts
```

### 6.4 테스트 격리성 유지

테스트 간 상태 공유를 방지하고 독립성을 보장하기 위해 상태 초기화 및 모킹 초기화가 필요합니다.

```ts
import { afterEach, vi } from 'vitest';
import { cleanup } from '@testing-library/react';

afterEach(() => {
  cleanup();
  vi.resetAllMocks();
});
```

### 6.5 명확한 Assertion 및 오류 메시지 사용

의미 있는 assertion을 사용해 실패 원인을 빠르게 파악할 수 있도록 합니다.

```tsx
expect(screen.getByRole('button', { name: '제출' })).toBeInTheDocument();
expect(screen.getByRole('button', { name: '제출' })).toHaveAttribute('disabled');
```

본 가이드를 통해 Next.js 환경에서 Vitest와 React Testing Library를 효과적으로 활용하여 실무 적용 능력을 높일 수 있습니다.


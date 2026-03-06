# Code Style Guide

## 기본 원칙

- ESLint + Prettier 설정을 따른다.
- 자동 포맷팅이 적용되므로 스타일 논쟁을 피한다.

## Import 순서

| 순서 | 대상 | 예시 |
|---|---|---|
| 1 | React | `import { useState, useEffect } from 'react'` |
| 2 | 외부 라이브러리 | `import { useQuery } from '@tanstack/react-query'` |
| 3 | 내부 모듈 (pages, shared) | `import { Button } from '@/shared/components'` |
| 4 | 타입 (type-only import) | `import type { Item } from '@/pages/{domain}'` |
| 5 | 스타일 | `import styles from './ItemList.module.css'` |

## 컴포넌트 내부 구조

| 순서 | 내용 |
|---|---|
| 1 | 타입 정의 (Props interface) |
| 2-1 | 훅 (useNavigate, useState 등) |
| 2-2 | 파생값 (useMemo 등) |
| 2-3 | 이벤트 핸들러 (handle*) |
| 2-4 | 렌더링 (return JSX) |
| 3 | export |

```tsx
interface ItemCardProps {
  item: Item;
  onEdit: (id: string) => void;
}

const ItemCard = ({ item, onEdit }: ItemCardProps) => {
  const navigate = useNavigate();
  const [isOpen, setIsOpen] = useState(false);

  const formattedPrice = formatPrice(item.price);

  const handleEdit = () => {
    onEdit(item.id);
  };

  return (
    <div>
      <h3>{item.name}</h3>
      <p>{formattedPrice}</p>
      <Button onClick={handleEdit}>수정</Button>
    </div>
  );
};

export { ItemCard };
```

## 금지 패턴

| 패턴 | Bad | Good |
|---|---|---|
| index.ts에 로직 작성 | `index.ts`에서 컴포넌트 정의 | 별도 파일 작성 후 `index.ts`에서 re-export |
| 콘솔 로그 | `console.log('data:', data)` | 디버깅 후 반드시 제거 |
| 매직 넘버 | `items.length > 10` | `const MAX_COUNT = 10; items.length > MAX_COUNT` |
| 인라인 스타일 남용 | `style={{ marginTop: 16 }}` | CSS Module 또는 클래스 사용 |

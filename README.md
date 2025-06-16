# 메모이제이션 함수 구현 및 React 훅과의 비교 분석

## 목차
1. [메모이제이션 함수 구현](#메모이제이션-함수-구현)
2. [useMemo와의 차이점](#useMemo와의-차이점)
3. [useCallback과의 차이점](#useCallback과의-차이점)
4. [종합 비교 및 결론](#종합-비교-및-결론)

---

## 메모이제이션 함수 구현

### 1. 기본 메모이제이션 함수

```typescript
// utils/memoization.ts
type MemoizedFunction<T extends (...args: any[]) => any> = T & {
  cache: Map<string, ReturnType<T>>;
  clearCache: () => void;
};

const createBasicMemoize = <T extends (...args: any[]) => any>(
  fn: T
): MemoizedFunction<T> => {
  const cache = new Map<string, ReturnType<T>>();

  const memoizedFn = ((...args: Parameters<T>): ReturnType<T> => {
    const cacheKey = JSON.stringify(args);
    
    if (cache.has(cacheKey)) {
      console.log('캐시에서 결과 반환:', cacheKey);
      return cache.get(cacheKey)!;
    }

    console.log('함수 실행 후 캐시 저장:', cacheKey);
    const result = fn(...args);
    cache.set(cacheKey, result);
    
    return result;
  }) as MemoizedFunction<T>;

  memoizedFn.cache = cache;
  memoizedFn.clearCache = (): void => {
    cache.clear();
    console.log('캐시가 클리어되었습니다.');
  };

  return memoizedFn;
};
```

### 2. 고급 메모이제이션 함수 (LRU 캐시 포함)

```typescript
// utils/advancedMemoization.ts
interface MemoizeOptions {
  maxCacheSize?: number;
  ttl?: number; // Time to live in milliseconds
  enableLRU?: boolean;
}

interface CacheEntry<T> {
  value: T;
  timestamp: number;
  accessCount: number;
  lastAccessed: number;
}

const createAdvancedMemoize = <T extends (...args: any[]) => any>(
  fn: T,
  options: MemoizeOptions = {}
): AdvancedMemoizedFunction<T> => {
  const {
    maxCacheSize = 100,
    ttl = 5 * 60 * 1000, // 5분 기본값
    enableLRU = true
  } = options;

  const cache = new Map<string, CacheEntry<ReturnType<T>>>();
  let totalHits = 0;
  let totalMisses = 0;

  // TTL 검사, LRU 정책, 캐시 통계 등의 고급 기능 구현
  // ... (상세 구현 코드)
};
```

### 3. React 전용 메모이제이션 훅

```typescript
// hooks/useMemoizedCallback.ts
const useMemoizedCallback = <T extends (...args: any[]) => any>(
  callback: T,
  deps: React.DependencyList = []
): MemoizedCallback<T> => {
  const cacheRef = useRef(new Map<string, ReturnType<T>>());

  const memoizedCallback = useCallback(
    ((...args: Parameters<T>): ReturnType<T> => {
      const cacheKey = JSON.stringify(args);
      
      if (cacheRef.current.has(cacheKey)) {
        return cacheRef.current.get(cacheKey)!;
      }

      const result = callback(...args);
      cacheRef.current.set(cacheKey, result);
      
      return result;
    }) as MemoizedCallback<T>,
    deps
  );

  return memoizedCallback;
};
```

---

## useMemo와의 차이점

### 핵심 차이점

| 특징 | useMemo | 커스텀 메모이제이션 |
|------|---------|-------------------|
| **주요 용도** | 컴포넌트 렌더링 최적화 | 함수 실행 최적화 |
| **캐시 지속성** | 컴포넌트 라이프사이클 | 명시적 해제까지 지속 |
| **사용 범위** | 컴포넌트 내부만 | 전역 어디서든 |
| **메모리 관리** | React 자동 관리 | 수동 관리 |
| **고급 기능** | 기본 메모이제이션만 | TTL, LRU, 통계 등 |

### 1. 용도와 목적

```typescript
// useMemo - 값(Value) 메모이제이션
const ExpensiveComponent: React.FC<{ items: Item[] }> = ({ items }) => {
  const expensiveValue = useMemo(() => {
    return items.reduce((sum, item) => sum + item.price, 0);
  }, [items]);

  return <div>총합: {expensiveValue}</div>;
};

// 커스텀 함수 - 함수(Function) 메모이제이션
const expensiveCalculation = (items: Item[]) => {
  return items.reduce((sum, item) => sum + item.price, 0);
};

const memoizedCalc = createAdvancedMemoize(expensiveCalculation);
```

### 2. 라이프사이클과 지속성

```typescript
// useMemo - 컴포넌트 라이프사이클에 종속
const ComponentA: React.FC = () => {
  const memoizedValue = useMemo(() => {
    return heavyCalculation(count);
  }, [count]); // 컴포넌트가 언마운트되면 캐시 사라짐
};

// 커스텀 메모이제이션 - 독립적인 캐시 관리
const memoizedHeavyCalc = createAdvancedMemoize(heavyCalculation);
// 컴포넌트가 언마운트되어도 캐시는 유지됨
// 다른 컴포넌트에서도 같은 캐시 공유 가능
```

---

## useCallback과의 차이점

### 핵심 차이점: 무엇을 메모이제이션하는가?

- **useCallback**: 함수 참조(Reference) 메모이제이션
- **커스텀 함수**: 함수 실행 결과(Result) 메모이제이션

### 1. 메모이제이션 대상

```typescript
// useCallback - 함수 참조를 메모이제이션
const handleClick = useCallback(() => {
  console.log('클릭!'); // 매번 실행됨
  doSomething();
}, [dependency]); // 의존성이 같으면 같은 함수 참조 반환

// 커스텀 메모이제이션 - 함수 실행 결과를 메모이제이션
const memoizedCalc = createAdvancedMemoize(expensiveCalculation);
const result = memoizedCalc(data); // 같은 data면 계산하지 않고 캐시된 결과 반환
```

### 2. 성능 최적화 관점

| 특징 | useCallback | 커스텀 메모이제이션 |
|------|-------------|-------------------|
| **메모이제이션 대상** | 함수 참조 | 함수 실행 결과 |
| **주요 목적** | 리렌더링 최적화 | 계산/API 호출 최적화 |
| **실행 여부** | 매번 실행됨 | 캐시 있으면 실행 안 됨 |
| **캐시 기준** | 의존성 배열 | 함수 인자값 |
| **성능 개선** | 렌더링 성능 | 계산 성능 |

### 3. 실제 사용 사례

```typescript
// useCallback - 컴포넌트 최적화
const TodoList: React.FC = () => {
  const handleToggle = useCallback((id: string) => {
    updateTodo(id); // 매번 실행됨
  }, []); // 함수 참조는 안정적

  return (
    <div>
      {todos.map(todo => (
        <TodoItem onToggle={handleToggle} /> // 리렌더링 방지
      ))}
    </div>
  );
};

// 커스텀 메모이제이션 - 계산 최적화
const memoizedTodoStats = createAdvancedMemoize(calculateTodoStats);

const TodoStats: React.FC = () => {
  const stats = memoizedTodoStats(todos); // 같은 todos면 계산 안 함
  
  return <div>완료: {stats.completed}</div>;
};
```

---

## 종합 비교 및 결론

### 전체 비교표

| 특징 | useMemo | useCallback | 커스텀 메모이제이션 |
|------|---------|-------------|-------------------|
| **메모이제이션 대상** | 계산된 값 | 함수 참조 | 함수 실행 결과 |
| **주요 목적** | 값 계산 최적화 | 리렌더링 최적화 | 함수 실행 최적화 |
| **캐시 지속성** | 컴포넌트 생명주기 | 컴포넌트 생명주기 | 명시적 해제까지 |
| **사용 범위** | 컴포넌트 내부 | 컴포넌트 내부 | 전역 |
| **의존성 관리** | 명시적 배열 | 명시적 배열 | 자동 (함수 인자) |
| **고급 기능** | 없음 | 없음 | TTL, LRU, 통계 등 |
| **메모리 관리** | React 자동 | React 자동 | 수동 |

### 사용 가이드라인

#### useMemo를 사용할 때:
- 컴포넌트 렌더링 중 값 계산 최적화
- 컴포넌트 내부에서만 사용하는 계산된 값
- React의 렌더링 라이프사이클과 연동 필요

```typescript
const filteredItems = useMemo(() => 
  items.filter(item => item.category === selectedCategory),
  [items, selectedCategory]
);
```

#### useCallback을 사용할 때:
- 자식 컴포넌트 리렌더링 방지
- 함수 참조의 안정성이 필요한 경우
- React.memo와 함께 사용하는 최적화

```typescript
const handleClick = useCallback(() => {
  doSomething();
}, [dependency]);
```

#### 커스텀 메모이제이션을 사용할 때:
- 비싼 계산이나 API 호출의 중복 실행 방지
- 여러 컴포넌트에서 공유하는 로직
- 고급 캐시 기능 (TTL, LRU 등) 필요
- 캐시 통계나 수동 관리 필요

```typescript
const memoizedApiCall = createAdvancedMemoize(fetchUserData, {
  ttl: 5 * 60 * 1000, // 5분
  maxCacheSize: 100
});
```

### 조합 사용 패턴

세 가지를 함께 사용하는 경우:

```typescript
const ComplexComponent: React.FC = () => {
  // 1. 계산된 값 메모이제이션
  const processedData = useMemo(() => 
    processData(rawData), [rawData]
  );

  // 2. API 호출 결과 메모이제이션 (전역 캐시)
  const memoizedFetch = useMemo(() => 
    createAdvancedMemoize(fetchData), []
  );

  // 3. 함수 참조 안정화
  const handleRefresh = useCallback(async () => {
    const data = await memoizedFetch(processedData);
    setRefreshedData(data);
  }, [processedData, memoizedFetch]);

  return <RefreshButton onClick={handleRefresh} />;
};
```

---

이번 과제에서는 최적화에 대해 깊이 있는 학습을 해볼 수 있었다.
특히 메모이제이션 관련 기법들(useMemo, useCallback, React.memo)을 다양한 맥락에서 적용해볼 수 있어서 좋았다.

처음에는 단순히 "렌더링 최적화" 정도만 생각하고 접근했지만, 막상 코드를 뜯어보니 렌더링 최적화가 단순히 컴포넌트를 감싸는 것만으로는 해결되지 않는 경우가 많다는 걸 느꼈다.

컴포넌트가 어떤 상태를 의존하고 있는지, 어떤 props가 변화하고 있는지, 내부에서 어떤 연산이 반복되고 있는지, context로 인해 전체 리렌더가 발생하고 있는 것은 아닌지 등을 하나하나 확인하면서 정말 많은 인사이트를 얻었다.

특히 이번 과제에서는 API 호출 최적화 → 렌더링 연산 최적화 → DnD 성능 최적화까지 단계별로 진행하면서 React 성능 최적화에서 고려해야 할 포인트가 무엇인지 많이 배울 수 있었다.

# SearchDialog 개선

## API 호출 최적화

### 기존 코드

```tsx
const fetchMajors = () => axios.get<Lecture[]>("/schedules-majors.json");
const fetchLiberalArts = () =>
  axios.get<Lecture[]>("/schedules-liberal-arts.json");

const fetchAllLectures = async () =>
  await Promise.all([
    (console.log("API Call 1", performance.now()), await fetchMajors()),
    (console.log("API Call 2", performance.now()), await fetchLiberalArts()),
    (console.log("API Call 3", performance.now()), await fetchMajors()),
    (console.log("API Call 4", performance.now()), await fetchLiberalArts()),
    (console.log("API Call 5", performance.now()), await fetchMajors()),
    (console.log("API Call 6", performance.now()), await fetchLiberalArts()),
  ]);

useEffect(() => {
  const start = performance.now();
  console.log("API 호출 시작: ", start);
  fetchAllLectures().then((results) => {
    const end = performance.now();
    console.log("모든 API 호출 완료 ", end);
    console.log("API 호출에 걸린 시간(ms): ", end - start);
    setLectures(results.flatMap((result) => result.data));
  });
}, []);
```

### 문제점

- 기존 API 요청마다 모두 호출
- promise.all의 잘못된 사용(직렬 -> 병렬로 변경 필요)

### 해결방안

- API 요청을 캐싱
- promise.all이 병렬로 처리하게 변경 (기존에는 배열을 만들 때 await가 있어 직렬로 처리되었음, promise 객체를 그대로 넣어줘야 병렬로 처리됨)

### 개선 코드

```tsx
import { useCallback, useRef } from "react";
import { Lecture } from "./types";
import axios from "axios";

export const useLectureFetcher = () => {
  const majorsCacheRef = useRef<Lecture[] | null>(null);
  const liberalArtsCacheRef = useRef<Lecture[] | null>(null);

  const fetchMajors = useCallback(async () => {
    if (majorsCacheRef.current) {
      return majorsCacheRef.current;
    }
    const response = await axios.get<Lecture[]>("/schedules-majors.json");
    majorsCacheRef.current = response.data;
    return majorsCacheRef.current;
  }, []);

  const fetchLiberalArts = useCallback(async () => {
    if (liberalArtsCacheRef.current) {
      return liberalArtsCacheRef.current;
    }
    const response = await axios.get<Lecture[]>("/schedules-liberal-arts.json");
    liberalArtsCacheRef.current = response.data;
    return liberalArtsCacheRef.current;
  }, []);

  const fetchAllLectures = useCallback(async () => {
    return await Promise.all([
      (console.log("API Call 1", performance.now()), fetchMajors()),
      (console.log("API Call 2", performance.now()), fetchLiberalArts()),
      (console.log("API Call 3", performance.now()), fetchMajors()),
      (console.log("API Call 4", performance.now()), fetchLiberalArts()),
      (console.log("API Call 5", performance.now()), fetchMajors()),
      (console.log("API Call 6", performance.now()), fetchLiberalArts()),
    ]);
  }, [fetchMajors, fetchLiberalArts]);

  return { fetchAllLectures };
};
```

- useRef를 사용해 캐싱
- Promise.all에 promise 객체를 그대로 넣어줘 병렬처리 가능하도록 변경

## 불필요한 연산 최적화

### 기존 문제

- `getFilteredLectures` 함수에서 불필요한 연산이 발생(`parseSchedule` 함수 중복 실행)
- 컴포넌트가 리렌더링 될 때마다 함수가 실행되어 성능 저하

### 해결방안

- 조건문을 변경해 `parseSchedule` 함수를 중복 실행하지 않도록 함
- `useMemo`를 사용해 불필요한 연산 최소화
- 함수를 외부로 빼서 컴포넌트 생명주기와 분리
- 해당 함수의 결과값을 사용하는 변수들에 대해서도 `useMemo` 적용

### 개선 코드

```ts
import { SearchOption } from "./SearchDialog";
import { Lecture } from "./types";
import { parseSchedule } from "./utils";

export const getFilteredLectures = (
  lectures: Lecture[],
  searchOptions: SearchOption
) => {
  const { query = "", credits, grades, days, times, majors } = searchOptions;

  return lectures
    .filter(
      (lecture) =>
        lecture.title.toLowerCase().includes(query.toLowerCase()) ||
        lecture.id.toLowerCase().includes(query.toLowerCase())
    )
    .filter((lecture) => grades.length === 0 || grades.includes(lecture.grade))
    .filter((lecture) => majors.length === 0 || majors.includes(lecture.major))
    .filter(
      (lecture) => !credits || lecture.credits.startsWith(String(credits))
    )
    .filter((lecture) => {
      if (days.length === 0 && times.length === 0) {
        return true;
      }

      const schedules = lecture.schedule ? parseSchedule(lecture.schedule) : [];

      const matchDay =
        days.length === 0 || schedules.some((s) => days.includes(s.day));

      const matchTime =
        times.length === 0 ||
        schedules.some((s) => s.range.some((time) => times.includes(time)));

      return matchDay && matchTime;
    });
};
```

```tsx
const filteredLectures = useMemo(() => {
  return getFilteredLectures(lectures, searchOptions);
}, [lectures, searchOptions]);

const lastPage = useMemo(() => {
  return Math.ceil(filteredLectures.length / PAGE_SIZE);
}, [filteredLectures]);

const visibleLectures = useMemo(() => {
  return filteredLectures.slice(0, page * PAGE_SIZE);
}, [filteredLectures, page]);

const allMajors = useMemo(() => {
  return [...new Set(lectures.map((lecture) => lecture.major))];
}, [lectures]);
```

## 불필요한 렌더링 최적화 (전공 목록)

### 기존 문제

- `MajorFilterSection` 컴포넌트에서 상위 컴포넌트의 상태 변경 시 하위 모든 전공 아이템들이 불필요하게 리렌더링되는 성능 이슈 발생
- React의 기본 렌더링 동작으로 인해 props가 변경되지 않은 자식 컴포넌트들도 부모 컴포넌트 리렌더링에 따라 재렌더링됨
- 전공 목록 배열(`allMajors`) 계산이 매 렌더링 사이클마다 실행되어 O(n) 시간복잡도 연산이 반복됨
- 이벤트 핸들러 함수(`toggleMajor`)가 매번 새로 생성되어 자식 컴포넌트에 새로운 참조값을 전달하며 불필요한 리렌더링 트리거

### 해결 방안

- `React.memo`를 활용하여 props가 실제로 변경될 때만 컴포넌트가 리렌더링되도록 최적화
- `useMemo`를 사용하여 전공 목록 계산을 메모이제이션하고 의존성이 변경될 때만 재계산
- `useCallback`으로 이벤트 핸들러 함수를 메모이제이션하여 자식 컴포넌트의 불필요한 리렌더링 방지

### 개선 내용

- 컴포넌트 메모이제이션: `MajorFilterSection`, `MajorCheckboxList`, `MajorItem` 컴포넌트를 `React.memo`로 래핑
- 전공 목록 메모이제이션: `allMajors` 계산을 `useMemo`로 최적화하여 `lectures` 데이터 변경 시에만 재계산
- 이벤트 핸들러 메모이제이션: `toggleMajor` 함수를 `useCallback`으로 메모이제이션하여 참조 안정성 확보
- 필터링 로직 분리: `getFilteredLectures` 함수를 컴포넌트 외부로 분리하여 재사용성과 성능 향상

```tsx
// 개선된 코드 예시
const allMajors = useMemo(() => {
  return [...new Set(lectures.map((lecture) => lecture.major))];
}, [lectures]);

const toggleMajor = useCallback((major: string) => {
  setSearchOptions((prev) => {
    const newMajors = prev.majors.includes(major)
      ? prev.majors.filter((m) => m !== major)
      : [...prev.majors, major];
    return { ...prev, majors: newMajors };
  });
  setPage(1);
  loaderWrapperRef.current?.scrollTo(0, 0);
}, []);

// 컴포넌트 메모이제이션 적용
export const MajorItem = React.memo(({ major, isSelected, onToggle }) => {
  return (
    <Box key={major}>
      <Checkbox
        size="sm"
        value={major}
        isChecked={isSelected}
        onChange={() => onToggle(major)}
      >
        {major.replace(/<p>/gi, " ")}
      </Checkbox>
    </Box>
  );
});
```

## 불필요한 렌더링 최적화 (강의 목록)

### 기존 문제

- 강의 목록 컴포넌트에서 인피니트 스크롤 시 기존 렌더링된 모든 강의 아이템들이 불필요하게 재렌더링됨
- 페이지네이션으로 새 데이터 로드 시 전체 목록이 다시 렌더링되어 성능 저하 발생

### 해결 방안

- `React.memo`를 통한 `LectureRow` 컴포넌트 메모이제이션으로 props 변경 시에만 리렌더링
- `useMemo`를 활용한 `visibleLectures` 계산 최적화로 필터링 결과와 페이지 번호 변경 시에만 재계산
- `useCallback`을 통한 이벤트 핸들러(`addSchedule`) 메모이제이션으로 자식 컴포넌트 재렌더링 방지
- 강의 ID 기반의 안정적인 `key` 속성 사용으로 React reconciliation 최적화

### 개선 내용

- LectureRow 메모이제이션: `React.memo`로 래핑하여 props가 변경될 때만 개별 강의 행이 리렌더링되도록 최적화
- 리스트 데이터 캐싱: `visibleLectures`를 `useMemo`로 메모이제이션하여 `filteredLectures`와 `page` 변경 시에만 재계산
- 이벤트 핸들러 최적화: `addSchedule` 함수를 `useCallback`으로 메모이제이션하여 참조 안정성 확보
- 키 최적화: `lecture.id`와 `index`를 조합한 안정적인 `key` 속성으로 불필요한 DOM 조작 최소화
- 필터링 로직 분리: `getFilteredLectures` 함수를 외부로 분리하여 컴포넌트 렌더링과 독립적으로 실행

```tsx
// 개선된 코드 예시
export const LectureRow = React.memo(
  ({ lecture, index, onAddSchedule }: LectureRowProps) => {
    return (
      <Tr key={`${lecture.id}-${index}`}>
        <Td width="100px">{lecture.id}</Td>
        <Td width="50px">{lecture.grade}</Td>
        <Td width="200px">{lecture.title}</Td>
        <Td width="50px">{lecture.credits}</Td>
        <Td width="150px" dangerouslySetInnerHTML={{ __html: lecture.major }} />
        <Td
          width="150px"
          dangerouslySetInnerHTML={{ __html: lecture.schedule }}
        />
        <Td width="80px">
          <Button
            size="sm"
            colorScheme="green"
            onClick={() => onAddSchedule(lecture)}
          >
            추가
          </Button>
        </Td>
      </Tr>
    );
  }
);

// SearchDialog 내부 최적화
const visibleLectures = useMemo(() => {
  return filteredLectures.slice(0, page * PAGE_SIZE);
}, [filteredLectures, page]);

const addSchedule = useCallback(
  (lecture: Lecture) => {
    if (!searchInfo) return;
    const { tableId } = searchInfo;
    const schedules = parseSchedule(lecture.schedule).map((schedule) => ({
      ...schedule,
      lecture,
    }));
    setSchedulesMap((prev) => ({
      ...prev,
      [tableId]: [...prev[tableId], ...schedules],
    }));
    onClose();
  },
  [onClose, searchInfo, setSchedulesMap]
);
```

# DND 시스템 개선

## 드래그시 렌더링 최적화

### 기존 문제

- 드래그를 할 때 거의 모든 요소가 리렌더링 됨
- DND 상태 변경 시 관련 없는 컴포넌트들까지 불필요하게 재렌더링되어 성능 저하 발생
- 드래그 중 실시간으로 발생하는 상태 업데이트로 인해 전체 컴포넌트 트리가 리렌더링됨
- 메모이제이션이 적절히 적용되지 않아 드래그 과정에서 끊김 현상과 성능 이슈 발생

### 해결 방안

- `useDndContext`를 활용하여 DND 상태를 필요한 컴포넌트에서만 구독하도록 상태관리 최적화
- `React.memo`를 통한 컴포넌트 메모이제이션으로 DND와 관련 없는 컴포넌트의 불필요한 리렌더링 방지
- `useMemo`와 `useCallback`을 활용하여 드래그 관련 계산과 이벤트 핸들러 최적화
- DND 상태를 세분화하여 필요한 부분만 업데이트되도록 상태 구조 개선
- 드래그 중인 요소와 드롭 가능한 요소만 선택적으로 리렌더링되도록 최적화

### 개선 내용

- DND Context 선택적 구독: `ScheduleTable`에서 `useDndContext`를 사용하여 드래그 중인 테이블만 활성화 표시
- DraggableSchedule 메모이제이션: 커스텀 비교 함수로 props 변경 시에만 리렌더링되도록 최적화
- 계산 로직 캐싱: `getColor`, `activeTableId`, `scheduleComponents` 등을 `useMemo`로 메모이제이션
- 이벤트 핸들러 최적화: `handleTimeClick`, `handleDragEnd` 등을 `useCallback`으로 메모이제이션
- 정적 컴포넌트 보호: `TableGrid`, `TableContainer` 등 드래그와 무관한 컴포넌트를 `React.memo`로 보호

```tsx
// 기존 ScheduleTable.tsx - 개선 전
const ScheduleTable = ({ tableId, schedules, onScheduleTimeClick, onDeleteButtonClick }) => {
  const dndContext = useDndContext(); // 전체 DND 상태 구독으로 불필요한 리렌더링 발생

  // 메모이제이션 없이 매번 재계산
  const getColor = (lectureId: string): string => {
    const lectures = [...new Set(schedules.map(({ lecture }) => lecture.id))];
    const colors = ["#fdd", "#ffd", "#dff", "#ddf", "#fdf", "#dfd"];
    return colors[lectures.indexOf(lectureId) % colors.length];
  };

  return (
    <TableContainer isActive={activeTableId === tableId}>
      <TableGrid onTimeClick={handleTimeClick} />
      {schedules.map((schedule, index) => (
        <DraggableSchedule key={`${schedule.lecture.id}-${index}`} ... />
      ))}
    </TableContainer>
  );
};

// 개선된 ScheduleTable.tsx - 개선 후
const ScheduleTable = ({ tableId, schedules, onScheduleTimeClick, onDeleteButtonClick }) => {
  const dndContext = useDndContext();

  // 색상 계산 메모이제이션 - schedules 변경 시에만 재계산
  const getColor = useCallback((lectureId: string): string => {
    const lectures = [...new Set(schedules.map(({ lecture }) => lecture.id))];
    const colors = ["#fdd", "#ffd", "#dff", "#ddf", "#fdf", "#dfd"];
    return colors[lectures.indexOf(lectureId) % colors.length];
  }, [schedules]);

  // 활성 테이블 ID만 추출하여 불필요한 리렌더링 방지
  const activeTableId = useMemo(() => {
    const activeId = dndContext.active?.id;
    if (activeId) {
      return String(activeId).split(":")[0];
    }
    return null;
  }, [dndContext.active?.id]);

  // 이벤트 핸들러 메모이제이션
  const handleTimeClick = useCallback((day: string, time: number) => {
    onScheduleTimeClick?.({ day, time });
  }, [onScheduleTimeClick]);

  // 스케줄 컴포넌트들 메모이제이션 - 의존성 변경 시에만 재생성
  const scheduleComponents = useMemo(() => {
    return schedules.map((schedule, index) => (
      <DraggableSchedule
        key={`${schedule.lecture.id}-${index}`}
        id={`${tableId}:${index}`}
        data={schedule}
        bg={getColor(schedule.lecture.id)}
        onDeleteButtonClick={() =>
          onDeleteButtonClick?.({ day: schedule.day, time: schedule.range[0] })
        }
      />
    ));
  }, [schedules, tableId, getColor, onDeleteButtonClick]);

  return (
    <TableContainer isActive={activeTableId === tableId}>
      <TableGrid onTimeClick={handleTimeClick} />
      {scheduleComponents}
    </TableContainer>
  );
};

// DraggableSchedule 컴포넌트 최적화
export const DraggableSchedule = React.memo(
  ({ id, data, bg, onDeleteButtonClick }: DraggableScheduleProps) => {
    // 위치 스타일 계산 메모이제이션
    const positionStyle = useMemo(() => ({
      position: "absolute" as const,
      left: `${120 + CellSize.WIDTH * leftIndex + 1}px`,
      top: `${40 + (topIndex * CellSize.HEIGHT + 1)}px`,
      width: CellSize.WIDTH - 1 + "px",
      height: CellSize.HEIGHT * size - 1 + "px",
      backgroundColor: bg,
      zIndex: isDragging ? 10 : 1,
    }), [leftIndex, topIndex, size, bg, isDragging]);

    // 변환 스타일 메모이제이션
    const transformStyle = useMemo(() =>
      CSS.Translate.toString(transform), [transform]
    );

    // 컨텐츠 메모이제이션
    const content = useMemo(() =>
      <ScheduleContent title={lecture.title} room={room} />,
      [lecture.title, room]
    );

    return (
      <Popover>
        <PopoverTrigger>
          <Box
            ref={setNodeRef}
            {...positionStyle}
            transform={transformStyle}
            {...listeners}
            {...attributes}
          >
            {content}
          </Box>
        </PopoverTrigger>
        {/* PopoverContent */}
      </Popover>
    );
  },
  // 커스텀 비교 함수로 필요한 props만 체크하여 리렌더링 최소화
  (prevProps, nextProps) => {
    if (prevProps.id !== nextProps.id) return false;
    const prevData = prevProps.data;
    const nextData = nextProps.data;
    return (
      prevProps.bg === nextProps.bg &&
      prevData.day === nextData.day &&
      prevData.room === nextData.room &&
      prevData.lecture.id === nextData.lecture.id &&
      prevData.lecture.title === nextData.lecture.title &&
      JSON.stringify(prevData.range) === JSON.stringify(nextData.range)
    );
  }
);

// 정적 컴포넌트 메모이제이션으로 불필요한 리렌더링 방지
export const TableGrid = React.memo(({ onTimeClick }: TableGridProps) => {
  return (
    <Grid templateColumns={...} templateRows={...}>
      <ScheduleTableHeader />
      {TIMES.map((time, timeIndex) => (
        <TimeRow
          key={`time-row-${timeIndex}`}
          timeIndex={timeIndex}
          time={time}
          onScheduleTimeClick={onTimeClick}
        />
      ))}
    </Grid>
  );
});

export const TableContainer = React.memo(({ isActive, children }: TableContainerProps) => {
  return (
    <Box
      position="relative"
      outline={isActive ? "5px dashed" : undefined}
      outlineColor="blue.300"
    >
      {children}
    </Box>
  );
});
```

## 드롭시 렌더링 최적화

### 기존 문제

- 모든 구간이 `schedulesMap`을 의존하고 있어 하나의 스케줄만 업데이트되어도 모든 컴포넌트가 리렌더링됨
- `schedulesMap`이 큰 덩어리로 관리되어 실제로는 한 개의 Schedule만 변경되었지만 모든 Schedule Data가 업데이트된 것으로 인식
- Context Provider에서 전체 상태를 하나의 객체로 관리하여 부분 업데이트가 불가능한 구조적 문제
- 각 `ScheduleTable`이 전체 `schedulesMap`을 구독하여 관련 없는 테이블도 함께 리렌더링되는 성능 이슈

### 해결 방안

- Context 분리 또는 Selector 패턴을 통한 개별 테이블 상태 구독 최적화
- `useMemo`와 `useCallback`을 활용하여 테이블별 데이터와 핸들러 메모이제이션
- 불변성을 유지하면서 최소한의 객체만 업데이트하는 상태 관리 개선
- 컴포넌트별 의존성을 최소화하여 관련 없는 렌더링 방지
- 개별 스케줄 테이블이 자신의 데이터만 구독하도록 Context 구조 개선

### 개선 내용

- Context 분리: `ScheduleProvider`를 개별 테이블별로 구독 가능하도록 분리하거나 selector 함수 도입
- 메모이제이션 적용: 각 테이블이 자신의 스케줄 데이터만 의존하도록 `useMemo`로 선택적 구독
- 상태 업데이트 최적화: `setSchedulesMap` 호출 시 변경된 테이블의 데이터만 업데이트되도록 최적화
- 컴포넌트 격리: `ScheduleTable` 컴포넌트가 자신과 관련된 상태 변경에만 반응하도록 의존성 최소화

```tsx
export const ScheduleProvider = ({ children }: PropsWithChildren) => {
  const [schedulesMap, setSchedulesMap] =
    useState<Record<string, Schedule[]>>(dummyScheduleMap);

  // 개별 테이블 데이터만 반환하는 선택자 함수
  const getSchedulesByTableId = useCallback(
    (tableId: string) => {
      return schedulesMap[tableId] || [];
    },
    [schedulesMap]
  );

  // 특정 테이블만 업데이트 - 다른 테이블은 참조 유지
  const updateSchedulesByTableId = useCallback((tableId: string, updater) => {
    setSchedulesMap((prev) => ({
      ...prev,
      [tableId]: updater(prev[tableId] || []),
    }));
  }, []);

  return (
    <ScheduleContext.Provider
      value={{
        schedulesMap,
        getSchedulesByTableId,
        updateSchedulesByTableId,
      }}
    >
      {children}
    </ScheduleContext.Provider>
  );
};

// 개별 테이블 컴포넌트 - 자신의 데이터만 구독
const ScheduleTableMemo = React.memo(
  ({ tableId, onScheduleTimeClick, onDeleteButtonClick }) => {
    const { getSchedulesByTableId } = useScheduleContext();

    // 해당 테이블의 스케줄만 구독 - 다른 테이블 변경 시 리렌더링 없음
    const schedules = useMemo(
      () => getSchedulesByTableId(tableId),
      [getSchedulesByTableId, tableId]
    );

    return (
      <ScheduleTable
        tableId={tableId}
        schedules={schedules}
        onScheduleTimeClick={onScheduleTimeClick}
        onDeleteButtonClick={onDeleteButtonClick}
      />
    );
  }
);

// DND 핸들러 - 특정 테이블만 업데이트
const handleDragEnd = useCallback(
  (event: any) => {
    const { active, delta } = event;
    const [tableId, index] = active.id.split(":");

    // 해당 테이블만 업데이트, 다른 테이블은 그대로 유지
    updateSchedulesByTableId(tableId, (schedules) => {
      const updatedSchedules = [...schedules];
      updatedSchedules[index] = {
        ...schedules[index],
        day: newDay,
        range: newRange,
      };
      return updatedSchedules;
    });
  },
  [updateSchedulesByTableId]
);
```

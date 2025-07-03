# 🎯 핵심 기능 명세

## 📋 개요

회의실 예약 시스템의 핵심 기능들을 상세히 정의하고, 각 기능의 구현 방법, 사용자 시나리오, 기술적 요구사항을 설명합니다. 모든 기능은 **사용자 중심 설계**와 **직관적인 UX**를 기반으로 구현됩니다.

## 🏢 1. 회의실 예약 관리

### 1.1 실시간 예약 현황 조회

#### 기능 설명
- 선택한 날짜 범위의 모든 예약 현황을 실시간으로 표시
- 회의실별, 시간대별 예약 상태를 시각적으로 확인
- 다양한 뷰 모드 지원 (일간/주간/월간)

#### 사용자 시나리오
```
1. 사용자가 예약 페이지에 접근
2. 기본적으로 오늘 날짜의 일간 뷰가 표시됨
3. 상단 네비게이션에서 날짜 범위와 뷰 모드 선택 가능
4. 실시간으로 다른 사용자의 예약 변경사항이 반영됨
5. 회의실 클릭 시 해당 회의실의 상세 정보 확인 가능
```

#### 기술적 구현
```typescript
// 예약 현황 조회 훅
export function useReservations(filters: ReservationFilters) {
  return useQuery({
    queryKey: ['reservations', filters],
    queryFn: () => reservationApiService.getReservations(filters),
    refetchInterval: 30000, // 30초마다 자동 갱신
    enabled: !!filters.start && !!filters.end,
  })
}

// 실시간 업데이트 처리
useEffect(() => {
  const handleReservationUpdate = (event: CustomEvent) => {
    queryClient.invalidateQueries(['reservations'])
  }
  
  window.addEventListener('reservationUpdated', handleReservationUpdate)
  return () => window.removeEventListener('reservationUpdated', handleReservationUpdate)
}, [])
```

#### UI/UX 요구사항
- **로딩 상태**: 스켈레톤 UI로 부드러운 로딩 경험
- **에러 처리**: 네트워크 오류 시 재시도 버튼 제공
- **반응형 디자인**: 모바일에서도 최적화된 뷰 제공
- **접근성**: 키보드 네비게이션 및 스크린 리더 지원

### 1.2 드래그 앤 드롭 예약 생성

#### 기능 설명
- 타임라인에서 드래그하여 새로운 예약을 직관적으로 생성
- 실시간 충돌 감지 및 시각적 피드백
- 30분 단위 스냅 기능으로 정확한 시간 설정

#### 사용자 시나리오
```
1. 사용자가 빈 시간대에서 마우스 드래그 시작
2. 드래그하는 동안 예약 블록이 실시간으로 표시됨
3. 다른 예약과 겹치는 경우 빨간색으로 표시되어 생성 불가
4. 드래그 완료 시 예약 생성 모달이 자동으로 열림
5. 기본 정보가 미리 입력된 상태로 제공됨
```

#### 기술적 구현
```typescript
// 드래그 상태 관리
const [dragState, setDragState] = useState<DragState>({
  isDragging: false,
  startTime: null,
  endTime: null,
  roomId: null,
  isValid: false,
})

// 드래그 이벤트 핸들러
const handleMouseDown = useCallback((event: MouseEvent, roomId: string) => {
  const startTime = calculateTimeFromPosition(event.clientY)
  setDragState({
    isDragging: true,
    startTime,
    endTime: startTime,
    roomId,
    isValid: true,
  })
}, [])

// 충돌 감지 로직
const checkConflicts = useMemo(() => {
  if (!dragState.startTime || !dragState.endTime) return []
  
  return existingReservations.filter(reservation => 
    reservation.roomId === dragState.roomId &&
    isTimeOverlapping(
      dragState.startTime!,
      dragState.endTime!,
      reservation.startTime,
      reservation.endTime
    )
  )
}, [dragState, existingReservations])
```

#### 드래그 시각적 피드백
- **유효한 드래그**: 파란색 반투명 블록
- **충돌 감지**: 빨간색 반투명 블록 + 경고 아이콘
- **스냅 가이드**: 30분 단위 격자선 표시
- **드래그 커서**: 전용 커서 아이콘 사용

### 1.3 예약 수정 및 삭제

#### 기능 설명
- 기존 예약 블록을 드래그하여 시간 변경
- 예약 상세 모달에서 모든 정보 수정 가능
- 권한 기반 수정/삭제 제어

#### 권한 체계
```typescript
const canEditReservation = (reservation: Reservation, currentUser: User) => {
  // 예약 생성자 본인
  if (reservation.createdBy === currentUser.id) return true
  
  // 관리자 권한
  if (currentUser.role === 'admin') return true
  
  // 부서 관리자 (같은 부서 예약)
  if (currentUser.role === 'department_admin' && 
      reservation.createdBy.departmentId === currentUser.departmentId) return true
  
  return false
}
```

#### 낙관적 업데이트
```typescript
const updateReservationMutation = useMutation({
  mutationFn: updateReservation,
  onMutate: async (updatedReservation) => {
    // 즉시 UI 업데이트
    await queryClient.cancelQueries(['reservations'])
    const previousReservations = queryClient.getQueryData(['reservations'])
    
    queryClient.setQueryData(['reservations'], (old: Reservation[]) =>
      old.map(r => r.id === updatedReservation.id ? updatedReservation : r)
    )
    
    return { previousReservations }
  },
  onError: (err, newReservation, context) => {
    // 에러 시 롤백
    queryClient.setQueryData(['reservations'], context?.previousReservations)
  },
})
```

## 🏗️ 2. 타임라인 뷰 시스템

### 2.1 다중 뷰 모드

#### 일간 뷰 (Day View)
- **시간 범위**: 08:00 ~ 22:00 (설정 가능)
- **시간 단위**: 30분 간격
- **회의실 표시**: 세로 레인으로 모든 회의실 표시
- **예약 블록**: 시간 비례 높이로 표시

```typescript
interface DayViewProps {
  date: Date
  rooms: MeetingRoom[]
  reservations: Reservation[]
  onReservationClick: (reservation: Reservation) => void
  onTimeSlotClick: (roomId: string, startTime: Date) => void
}
```

#### 주간 뷰 (Week View)
- **날짜 범위**: 7일간 (월요일 시작)
- **레이아웃**: 날짜별 세로 칼럼
- **예약 표시**: 간소화된 블록 형태
- **네비게이션**: 주 단위 이동 버튼

```typescript
interface WeekViewProps {
  startDate: Date // 주의 시작일 (월요일)
  rooms: MeetingRoom[]
  reservations: Reservation[]
  selectedRoomId?: string
}
```

#### 월간 뷰 (Month View)
- **달력 형태**: 전통적인 월간 달력 레이아웃
- **예약 표시**: 날짜별 예약 개수 및 간단한 제목
- **상세 보기**: 날짜 클릭 시 해당 일의 상세 뷰

```typescript
interface MonthViewProps {
  month: Date
  reservations: Reservation[]
  onDateClick: (date: Date) => void
}
```

### 2.2 실시간 동기화

#### WebSocket 이벤트 처리
```typescript
// WebSocket 메시지 타입
type WebSocketEvent = 
  | { type: 'reservation_created'; data: Reservation }
  | { type: 'reservation_updated'; data: Reservation }
  | { type: 'reservation_deleted'; data: { id: string } }
  | { type: 'room_status_changed'; data: { roomId: string; status: RoomStatus } }

// 이벤트 핸들러
const handleWebSocketMessage = (event: WebSocketEvent) => {
  switch (event.type) {
    case 'reservation_created':
      queryClient.setQueryData(['reservations'], (old: Reservation[]) => 
        [...old, event.data]
      )
      showToast('새로운 예약이 생성되었습니다', 'info')
      break
      
    case 'reservation_updated':
      queryClient.setQueryData(['reservations'], (old: Reservation[]) =>
        old.map(r => r.id === event.data.id ? event.data : r)
      )
      break
      
    case 'reservation_deleted':
      queryClient.setQueryData(['reservations'], (old: Reservation[]) =>
        old.filter(r => r.id !== event.data.id)
      )
      break
  }
}
```

#### 충돌 해결 전략
```typescript
// 동시 편집 감지
const detectConflict = (localReservation: Reservation, serverReservation: Reservation) => {
  return localReservation.updatedAt < serverReservation.updatedAt
}

// 충돌 해결 UI
const ConflictResolutionModal = ({ localData, serverData, onResolve }: Props) => {
  return (
    <Modal>
      <h3>예약 충돌이 감지되었습니다</h3>
      <div className="conflict-comparison">
        <div className="local-version">
          <h4>내 변경사항</h4>
          <ReservationPreview data={localData} />
          <Button onClick={() => onResolve('local')}>내 변경사항 적용</Button>
        </div>
        <div className="server-version">
          <h4>서버 최신 버전</h4>
          <ReservationPreview data={serverData} />
          <Button onClick={() => onResolve('server')}>서버 버전 적용</Button>
        </div>
      </div>
    </Modal>
  )
}
```

## 👥 3. 사용자 및 권한 관리

### 3.1 역할 기반 접근 제어 (RBAC)

#### 권한 레벨
```typescript
enum UserRole {
  GUEST = 'guest',           // 읽기 전용
  USER = 'user',             // 기본 사용자
  DEPARTMENT_ADMIN = 'dept_admin', // 부서 관리자
  ADMIN = 'admin',           // 시스템 관리자
  SUPER_ADMIN = 'super_admin' // 최고 관리자
}

interface Permission {
  resource: string
  action: 'create' | 'read' | 'update' | 'delete'
  condition?: (user: User, resource: any) => boolean
}
```

#### 권한 매트릭스
| 역할 | 예약 생성 | 예약 수정 | 예약 삭제 | 회의실 관리 | 사용자 관리 |
|------|-----------|-----------|-----------|-------------|-------------|
| **Guest** | ❌ | ❌ | ❌ | ❌ | ❌ |
| **User** | ✅ | ✅ (본인) | ✅ (본인) | ❌ | ❌ |
| **Dept Admin** | ✅ | ✅ (부서) | ✅ (부서) | ❌ | ❌ |
| **Admin** | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Super Admin** | ✅ | ✅ | ✅ | ✅ | ✅ |

#### 권한 체크 구현
```typescript
// 권한 체크 훅
export function usePermissions() {
  const { currentUser } = useAuthStore()
  
  const hasPermission = useCallback((
    resource: string, 
    action: string, 
    target?: any
  ): boolean => {
    if (!currentUser) return false
    
    const permission = getPermission(currentUser.role, resource, action)
    if (!permission) return false
    
    if (permission.condition && target) {
      return permission.condition(currentUser, target)
    }
    
    return true
  }, [currentUser])
  
  return { hasPermission }
}

// 컴포넌트에서 사용
const ReservationCard = ({ reservation }: Props) => {
  const { hasPermission } = usePermissions()
  
  const canEdit = hasPermission('reservation', 'update', reservation)
  const canDelete = hasPermission('reservation', 'delete', reservation)
  
  return (
    <Card>
      {/* 예약 정보 */}
      <div className="actions">
        {canEdit && <Button onClick={handleEdit}>수정</Button>}
        {canDelete && <Button onClick={handleDelete}>삭제</Button>}
      </div>
    </Card>
  )
}
```

### 3.2 조직도 기반 사용자 관리

#### 조직 구조
```typescript
interface Organization {
  id: string
  name: string
  code: string
  parentId: string | null
  level: number
  type: 'company' | 'division' | 'department' | 'team'
  children?: Organization[]
}

interface User {
  id: string
  name: string
  email: string
  role: UserRole
  organizationId: string
  organization: Organization
  position: string
  avatar?: string
}
```

#### 조직도 트리 컴포넌트
```typescript
const OrganizationTree = ({ onUserSelect }: Props) => {
  const { data: organizations } = useQuery(['organizations'], fetchOrganizations)
  
  const renderNode = (org: Organization) => (
    <TreeNode key={org.id} title={org.name}>
      {org.children?.map(renderNode)}
      {org.users?.map(user => (
        <TreeNode 
          key={user.id} 
          title={user.name}
          isLeaf
          onClick={() => onUserSelect(user)}
        />
      ))}
    </TreeNode>
  )
  
  return (
    <Tree>
      {organizations?.map(renderNode)}
    </Tree>
  )
}
```

## 🔔 4. 알림 시스템

### 4.1 실시간 알림

#### 알림 유형
```typescript
enum NotificationType {
  RESERVATION_REMINDER = 'reservation_reminder',    // 예약 알림
  RESERVATION_CONFLICT = 'reservation_conflict',    // 충돌 알림
  RESERVATION_CANCELLED = 'reservation_cancelled',  // 취소 알림
  ROOM_DISABLED = 'room_disabled',            // 회의실 점검
  SYSTEM_NOTICE = 'system_notice',                  // 시스템 공지
}

interface Notification {
  id: string
  type: NotificationType
  title: string
  message: string
  data?: Record<string, any>
  read: boolean
  createdAt: string
  userId: string
}
```

#### 알림 표시 컴포넌트
```typescript
const NotificationCenter = () => {
  const { notifications, markAsRead, markAllAsRead } = useNotificationStore()
  const unreadCount = notifications.filter(n => !n.read).length
  
  return (
    <Popover>
      <PopoverTrigger>
        <Button variant="ghost" size="icon">
          <Bell />
          {unreadCount > 0 && (
            <Badge className="notification-badge">{unreadCount}</Badge>
          )}
        </Button>
      </PopoverTrigger>
      
      <PopoverContent className="notification-panel">
        <div className="notification-header">
          <h3>알림</h3>
          {unreadCount > 0 && (
            <Button variant="ghost" size="sm" onClick={markAllAsRead}>
              모두 읽음
            </Button>
          )}
        </div>
        
        <ScrollArea className="notification-list">
          {notifications.map(notification => (
            <NotificationItem
              key={notification.id}
              notification={notification}
              onClick={() => markAsRead(notification.id)}
            />
          ))}
        </ScrollArea>
      </PopoverContent>
    </Popover>
  )
}
```

### 4.2 이메일 및 푸시 알림

#### 알림 설정
```typescript
interface NotificationSettings {
  emailNotifications: boolean
  pushNotifications: boolean
  reminderTiming: number[]     // 분 단위 [5, 15, 30]
  notificationTypes: NotificationType[]
  quietHours: {
    enabled: boolean
    start: string  // "22:00"
    end: string    // "08:00"
  }
}

const NotificationSettingsForm = () => {
  const [settings, setSettings] = useState<NotificationSettings>()
  
  return (
    <Form>
      <FormField name="emailNotifications">
        <Switch />
        <Label>이메일 알림 받기</Label>
      </FormField>
      
      <FormField name="reminderTiming">
        <Label>미리 알림 시간</Label>
        <CheckboxGroup>
          <Checkbox value={5}>5분 전</Checkbox>
          <Checkbox value={15}>15분 전</Checkbox>
          <Checkbox value={30}>30분 전</Checkbox>
        </CheckboxGroup>
      </FormField>
      
      <FormField name="quietHours">
        <Label>방해 금지 시간</Label>
        <TimeRangePicker />
      </FormField>
    </Form>
  )
}
```

## 📱 5. PWA 기능

### 5.1 오프라인 지원

#### 서비스 워커 전략
```typescript
// 캐시 전략
const CACHE_STRATEGIES = {
  // 정적 리소스: 캐시 우선
  static: 'CacheFirst',
  
  // API 데이터: 네트워크 우선, 캐시 폴백
  api: 'NetworkFirst',
  
  // 이미지: 캐시 우선, 네트워크 폴백
  images: 'CacheFirst',
  
  // 예약 데이터: 네트워크 전용 (실시간 중요)
  reservations: 'NetworkOnly',
}

// 오프라인 상태 감지
const useOfflineStatus = () => {
  const [isOffline, setIsOffline] = useState(!navigator.onLine)
  
  useEffect(() => {
    const handleOnline = () => setIsOffline(false)
    const handleOffline = () => setIsOffline(true)
    
    window.addEventListener('online', handleOnline)
    window.addEventListener('offline', handleOffline)
    
    return () => {
      window.removeEventListener('online', handleOnline)
      window.removeEventListener('offline', handleOffline)
    }
  }, [])
  
  return isOffline
}
```

#### 오프라인 UI
```typescript
const OfflineBanner = () => {
  const isOffline = useOfflineStatus()
  
  if (!isOffline) return null
  
  return (
    <div className="offline-banner">
      <WifiOff className="icon" />
      <span>오프라인 상태입니다. 일부 기능이 제한될 수 있습니다.</span>
    </div>
  )
}
```

### 5.2 앱 설치 및 업데이트

#### 설치 프롬프트
```typescript
const InstallPrompt = () => {
  const [deferredPrompt, setDeferredPrompt] = useState<any>(null)
  const [showInstallButton, setShowInstallButton] = useState(false)
  
  useEffect(() => {
    const handleBeforeInstallPrompt = (e: Event) => {
      e.preventDefault()
      setDeferredPrompt(e)
      setShowInstallButton(true)
    }
    
    window.addEventListener('beforeinstallprompt', handleBeforeInstallPrompt)
    
    return () => {
      window.removeEventListener('beforeinstallprompt', handleBeforeInstallPrompt)
    }
  }, [])
  
  const handleInstallClick = async () => {
    if (!deferredPrompt) return
    
    deferredPrompt.prompt()
    const { outcome } = await deferredPrompt.userChoice
    
    if (outcome === 'accepted') {
      setShowInstallButton(false)
    }
    
    setDeferredPrompt(null)
  }
  
  if (!showInstallButton) return null
  
  return (
    <Button onClick={handleInstallClick} className="install-button">
      <Download className="icon" />
      앱 설치하기
    </Button>
  )
}
```

## 🔍 6. 검색 및 필터링

### 6.1 통합 검색

#### 검색 기능
```typescript
interface SearchFilters {
  query: string           // 통합 검색어
  roomIds: string[]      // 회의실 필터
  dateRange: DateRange   // 날짜 범위
  timeRange: TimeRange   // 시간 범위
  status: ReservationStatus[]
  attendees: string[]    // 참석자 필터
  tags: string[]         // 태그 필터
}

const useSearch = () => {
  const [filters, setFilters] = useState<SearchFilters>({
    query: '',
    roomIds: [],
    dateRange: { start: new Date(), end: addDays(new Date(), 7) },
    timeRange: { start: '09:00', end: '18:00' },
    status: [],
    attendees: [],
    tags: [],
  })
  
  const { data: results, isLoading } = useQuery(
    ['search', filters],
    () => searchReservations(filters),
    { enabled: !!filters.query || filters.roomIds.length > 0 }
  )
  
  return { filters, setFilters, results, isLoading }
}
```

#### 고급 검색 UI
```typescript
const AdvancedSearchModal = ({ isOpen, onClose }: Props) => {
  const { filters, setFilters } = useSearch()
  
  return (
    <Modal isOpen={isOpen} onClose={onClose}>
      <ModalHeader>고급 검색</ModalHeader>
      
      <ModalBody>
        <SearchInput
          placeholder="예약 제목, 참석자, 설명 검색..."
          value={filters.query}
          onChange={(query) => setFilters(prev => ({ ...prev, query }))}
        />
        
        <RoomMultiSelect
          value={filters.roomIds}
          onChange={(roomIds) => setFilters(prev => ({ ...prev, roomIds }))}
        />
        
        <DateRangePicker
          value={filters.dateRange}
          onChange={(dateRange) => setFilters(prev => ({ ...prev, dateRange }))}
        />
        
        <TimeRangePicker
          value={filters.timeRange}
          onChange={(timeRange) => setFilters(prev => ({ ...prev, timeRange }))}
        />
      </ModalBody>
      
      <ModalFooter>
        <Button variant="outline" onClick={onClose}>취소</Button>
        <Button onClick={handleSearch}>검색</Button>
      </ModalFooter>
    </Modal>
  )
}
```

### 6.2 스마트 필터링

#### 자동 완성 검색
```typescript
const SmartSearchInput = () => {
  const [query, setQuery] = useState('')
  const [suggestions, setSuggestions] = useState<SearchSuggestion[]>([])
  
  const debouncedQuery = useDebounce(query, 300)
  
  useEffect(() => {
    if (debouncedQuery.length >= 2) {
      fetchSearchSuggestions(debouncedQuery).then(setSuggestions)
    } else {
      setSuggestions([])
    }
  }, [debouncedQuery])
  
  return (
    <Combobox value={query} onChange={setQuery}>
      <ComboboxInput placeholder="검색어를 입력하세요..." />
      
      <ComboboxOptions>
        {suggestions.map(suggestion => (
          <ComboboxOption key={suggestion.id} value={suggestion.text}>
            <div className="suggestion-item">
              <Icon name={suggestion.type} />
              <span>{suggestion.text}</span>
              <Badge>{suggestion.category}</Badge>
            </div>
          </ComboboxOption>
        ))}
      </ComboboxOptions>
    </Combobox>
  )
}
```

## 📊 7. 대시보드 및 통계

### 7.1 개인 대시보드

#### 주요 위젯
```typescript
const PersonalDashboard = () => {
  const { currentUser } = useAuthStore()
  const { data: myReservations } = useQuery(['my-reservations'], 
    () => getMyReservations(currentUser?.id)
  )
  
  return (
    <div className="dashboard-grid">
      <UpcomingReservationsWidget reservations={myReservations} />
      <QuickBookingWidget />
      <FavoriteRoomsWidget />
      <RecentActivityWidget />
      <UsageStatisticsWidget />
    </div>
  )
}

// 다가오는 예약 위젯
const UpcomingReservationsWidget = ({ reservations }: Props) => {
  const upcomingReservations = reservations
    ?.filter(r => new Date(r.startTime) > new Date())
    ?.slice(0, 5)
  
  return (
    <Card>
      <CardHeader>
        <h3>다가오는 예약</h3>
      </CardHeader>
      <CardBody>
        {upcomingReservations?.map(reservation => (
          <ReservationSummaryCard key={reservation.id} reservation={reservation} />
        ))}
      </CardBody>
    </Card>
  )
}
```

### 7.2 관리자 대시보드

#### 시스템 통계
```typescript
const AdminDashboard = () => {
  const { data: stats } = useQuery(['admin-stats'], fetchAdminStats)
  
  return (
    <div className="admin-dashboard">
      <StatsOverview stats={stats} />
      <RoomUtilizationChart data={stats?.roomUtilization} />
      <PopularTimesChart data={stats?.popularTimes} />
      <UserActivityChart data={stats?.userActivity} />
      <SystemHealthWidget />
    </div>
  )
}

// 회의실 이용률 차트
const RoomUtilizationChart = ({ data }: Props) => {
  return (
    <Card>
      <CardHeader>
        <h3>회의실 이용률</h3>
      </CardHeader>
      <CardBody>
        <ResponsiveContainer width="100%" height={300}>
          <BarChart data={data}>
            <XAxis dataKey="roomName" />
            <YAxis />
            <Tooltip />
            <Bar dataKey="utilizationRate" fill="#3b82f6" />
          </BarChart>
        </ResponsiveContainer>
      </CardBody>
    </Card>
  )
}
```

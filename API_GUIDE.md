# MeetRoom API 가이드

> GrooveSoft 회의실 예약 시스템 API 명세서 (버전 1.0.0)

## 기본 정보

### 서버 URL

-   **운영 서버**: `https://api.groovesoft-meeting.com`
-   **개발 서버 (Mock)**: `http://localhost:3000/api`

### 인증

-   **인증 방식**: JWT Bearer Token
-   **요청 헤더**: `Authorization: Bearer {YOUR_ACCESS_TOKEN}`

### 인증 및 상태 관리 (PWA 기준)

본 어플리케이션은 PWA 환경에 최적화된 토큰 기반 인증 및 상태 관리 전략을 따릅니다.

#### 1. 로그인 및 토큰 저장
- 사용자가 `POST /auth/login`을 통해 로그인에 성공하면, 서버는 **JWT (Access Token)** 를 발급합니다.
- 클라이언트(PWA)는 이 토큰을 `localStorage` 또는 `IndexedDB`와 같은 영구 저장소에 안전하게 저장합니다.

#### 2. 인증된 API 요청
- 로그인이 필요한 모든 API 요청(`GET /users/me` 등)의 `Authorization` 헤더에 저장된 토큰을 `Bearer {TOKEN}` 형식으로 포함하여 전송합니다.

#### 3. PWA 재실행 및 로그인 상태 유지
1. 사용자가 PWA를 다시 실행하면, 애플리케이션은 먼저 영구 저장소에 토큰이 있는지 확인합니다.
2. 토큰이 존재할 경우, 즉시 메인 화면을 보여주면서 백그라운드에서 `GET /users/me` API를 호출하여 토큰의 유효성을 검증하고 최신 사용자 정보를 가져옵니다.
3. 만약 API 호출이 `401 Unauthorized` 에러를 반환하면 (토큰 만료 등), 저장된 토큰을 삭제하고 사용자를 로그인 화면으로 리디렉션합니다.
4. 토큰이 없을 경우, 바로 로그인 화면으로 이동합니다.

#### 4. 데이터 로딩
- 조직도와 같은 대용량 데이터는 로그인 시점이나 앱 실행 시점에 **별도의 API (`GET /organization/departments` 등)를 통해 서버로부터 직접 가져옵니다.** (IndexedDB에 미리 저장해두는 방식이 아님)

---

## API 엔드포인트

### Authentication

#### `POST /auth/login`

이메일과 패스워드로 로그인하여 JWT 토큰을 발급받습니다.

-   **Request Body**:
    -   `email` (string, required)
    -   `password` (string, required)
    -   `organizationCode` (string, required)
-   **Response (200)**: [`LoginResponse`](#loginresponse)

---

### Users

#### `GET /users/me`

현재 로그인된 사용자의 상세 정보를 조회합니다. (인증 필요)

-   **Response (200)**: [`User`](#user)

#### `PUT /users/me`

현재 로그인된 사용자의 정보를 수정합니다. (인증 필요)

-   **Request Body**: [`UpdateUserRequest`](#updateuserrequest)
-   **Response (200)**: [`User`](#user)

---

### Rooms

#### `GET /rooms`

회의실 목록을 조회합니다.

-   **Query Parameters**:
    -   `floor` (integer, optional)
    -   `capacity` (integer, optional)
    -   `equipment` (string, optional): 쉼표로 구분된 장비 목록
-   **Response (200)**: Array of [`MeetingRoom`](#meetingroom)

#### `GET /rooms/{roomId}`

특정 회의실의 상세 정보를 조회합니다.

-   **Path Parameters**:
    -   `roomId` (string, required)
-   **Response (200)**: [`MeetingRoom`](#meetingroom)

#### `GET /rooms/{roomId}/availability`

특정 날짜의 회의실 예약 현황(가용 시간 포함)을 조회합니다. (모바일)

-   **Path Parameters**:
    -   `roomId` (string, required)
-   **Query Parameters**:
    -   `date` (string, required, format: date)
-   **Response (200)**: [`MeetingRoomAvailability`](#meetingroomavailability)

---

### Reservations

#### `GET /reservations`

예약 목록을 조회합니다. (인증 필요)

-   **Query Parameters**:
    -   `startDate` (string, required, format: date)
    -   `endDate` (string, required, format: date)
    -   `roomId` (string, optional)
    -   `userId` (string, optional)
-   **Response (200)**: Array of [`Reservation`](#reservation)

#### `POST /reservations`

새로운 예약을 생성합니다. (인증 필요)

-   **Request Body**: [`CreateReservationRequest`](#createreservationrequest)
-   **Response (201)**: [`Reservation`](#reservation)

#### `GET /reservations/{id}`

특정 예약의 상세 정보를 조회합니다. (인증 필요)

-   **Path Parameters**:
    -   `id` (string, required)
-   **Response (200)**: [`Reservation`](#reservation)

#### `PATCH /reservations/{id}`

기존 예약을 부분 수정합니다. (인증 필요)

-   **Path Parameters**:
    -   `id` (string, required)
-   **Request Body**: [`UpdateReservationRequest`](#updatereservationrequest-1)
-   **Response (200)**: [`Reservation`](#reservation)

#### `DELETE /reservations/{id}`

예약을 삭제합니다. (인증 필요)

-   **Path Parameters**:
    -   `id` (string, required)
-   **Response (204)**: No Content

---

### Organization

#### `GET /organization/departments`
모든 최상위 부서 목록을 조회합니다.

- **Response (200)**: 배열 [`Department`](#department)

#### `GET /organization/departments/{departmentId}/members`
특정 부서(ID) 및 하위 부서에 소속된 조직원과 서브 부서를 조회합니다.

- **Path Parameters**:
  - `departmentId` (string, required)
- **Response (200)**: 객체
  - `members`: [`OrganizationMember`](#organizationmember) 배열
  - `subDepartments`: [`Department`](#department) 배열

#### `GET /organization/members/search`
조직원(이름·이메일) 또는 부서명을 검색합니다.

- **Query Parameters**:
  - `q` (string, required) – 검색어
- **Response (200)**: 배열 [`OrganizationMember`](#organizationmember)

---

### Notifications

#### `GET /notifications`
사용자의 알림 목록을 조회합니다. (인증 필요)

- **Query Parameters**: (없음)
- **Response (200)**: 배열 [`Notification`](#notification)

#### `GET /notifications/unread`
읽지 않은 알림만 조회합니다. (인증 필요)

- **Response (200)**: 배열 [`Notification`](#notification)

#### `GET /notifications/recent`
최근 24시간(또는 서버 설정 기간) 내 알림을 조회합니다. (인증 필요)

- **Response (200)**: 배열 [`Notification`](#notification)

#### `PATCH /notifications/{id}/read`
특정 알림을 읽음 처리합니다. (인증 필요)

- **Path Parameters**:
  - `id` (string, required)
- **Response (200)**: No Content

#### `DELETE /notifications/{id}`
특정 알림을 삭제합니다. (인증 필요)

- **Path Parameters**:
  - `id` (string, required)
- **Response (204)**: No Content

#### `DELETE /notifications`
모든 알림을 삭제(비우기)합니다. (인증 필요)

- **Response (204)**: No Content

---

### Notices (공지사항)

#### `GET /notices`
공지사항 목록을 조회합니다.

- **Query Parameters**:
  - `page` (integer, optional, default=1)
  - `limit` (integer, optional, default=20)
- **Response (200)**: 배열 [`Notice`](#notice)

#### `GET /notices/{id}`
공지사항 상세를 조회합니다.

- **Path Parameters**:
  - `id` (string, required)
- **Response (200)**: [`Notice`](#notice)

---

## 데이터 모델 (Schemas)

### User
| 필드 | 타입 | 설명 |
|---|---|---|
| id | string | 사용자 ID |
| name | string | 이름 |
| email | string | 이메일 |
| department| string | 부서 |
| position | string | 직책 |
| preferences| [`UserPreferences`](#userpreferences) | 개인화 설정 |

### UpdateUserRequest
| 필드 | 타입 | 설명 |
|---|---|---|
| name | string | 이름 |
| phone | string | 연락처 |
| preferences| [`UserPreferences`](#userpreferences) | 개인화 설정 |

### LoginResponse
| 필드 | 타입 | 설명 |
|---|---|---|
| token | string | JWT 액세스 토큰 |
| user | [`User`](#user) | 사용자 정보 |

### MeetingRoom
| 필드 | 타입 | 설명 |
|---|---|---|
| id | string | 회의실 ID |
| name | string | 회의실 이름 |
| capacity | integer | 수용 인원 |
| floor | integer | 층 |
| equipment | array (string) | 보유 장비 |
| status | string | 상태 (`available`, `occupied`, `disabled`) |

### MeetingRoomAvailability
| 필드 | 타입 | 설명 |
|---|---|---|
| roomId | string | 회의실 ID |
| date | string (date) | 조회 날짜 |
| reservations | array ([`Reservation`](#reservation)) | 해당 날짜 예약 목록 |

### Reservation
| 필드 | 타입 | 설명 |
|---|---|---|
| id | string | 예약 ID |
| roomId | string | 회의실 ID |
| userId | string | 예약자 ID |
| userName | string | 예약자 이름 |
| title | string | 예약 제목 |
| description| string | 설명 |
| startTime | string (date-time) | 시작 시간 |
| endTime | string (date-time) | 종료 시간 |
| attendees | array ([`Attendee`](#attendee)) | 참석자 목록 |
| status | string | 상태 (`confirmed`, `cancelled`, `completed`) |
| createdAt | string (date-time) | 생성 시각 |
| updatedAt | string (date-time) | 마지막 수정 시각 |

### CreateReservationRequest
| 필드 | 타입 | 설명 |
|---|---|---|
| roomId | string | **(필수)** 회의실 ID |
| title | string | **(필수)** 예약 제목 |
| description| string | 설명 |
| startTime | string (date-time) | **(필수)** 시작 시간 |
| endTime | string (date-time) | **(필수)** 종료 시간 |
| attendees | array ([`CreateAttendeeRequest`](#createattendeerequest)) | 참석자 목록 |

### UpdateReservationRequest
| 필드 | 타입 | 설명 |
|---|---|---|
| title | string | 예약 제목 |
| description| string | 설명 |
| startTime | string (date-time) | 시작 시간 |
| endTime | string (date-time) | 종료 시간 |
| attendees | array ([`CreateAttendeeRequest`](#createattendeerequest)) | 참석자 목록 |
| status | string | 상태 (`confirmed`, `cancelled`, `completed`) |

### Attendee
| 필드 | 타입 | 설명 |
|---|---|---|
| id | string | 참석자 ID |
| name | string | 이름 |
| email | string | 이메일 |
| status | string | 상태 (`accepted`, `declined`) |

### CreateAttendeeRequest
| 필드 | 타입 | 설명 |
|---|---|---|
| name | string | 이름 |
| email | string | 이메일 |

### UserPreferences
| 필드 | 타입 | 설명 |
|---|---|---|
| theme | object | 테마 설정 |
| L mode | string | `light`, `dark`, `system` 중 하나 |
| favoriteRoomIds | array (string) | 선호하는 회의실 ID 목록 |
| reservationTemplates | array (object) | 예약 템플릿 목록 |
| L templateName | string | 템플릿 이름 |
| L title | string | 예약 제목 |
| L description| string | 설명 |
| L attendees | array (object) | 기본 참석자 (name, email) |

### Department
| 필드 | 타입 | 설명 |
|---|---|---|
| code | string | 부서 코드 |
| name | string | 부서명 |
| parent | string \| null | 상위 부서 코드(없으면 null) |
| hasChild | string | 하위 부서 존재 여부 (`"0"` 또는 자식 수) |

### OrganizationMember
| 필드 | 타입 | 설명 |
|---|---|---|
| id | string | 직원 ID |
| name | string | 이름 |
| email | string | 이메일 |
| position | string | 직책 |
| departmentId | string | 소속 부서 코드 |
| departmentName | string | 소속 부서명 |

### OrganizationTreeNode {#organizationtreenode}
| 필드 | 타입 | 설명 |
|---|---|---|
| id | string | 노드 ID |
| name | string | 이름(부서명/직원명) |
| type | string | `department` \| `person` |
| parentId | string | 상위 부서 ID (루트면 생략) |
| level | integer | 계층 깊이(0=루트) |
| position | string | 개인 직책 (person일 때) |
| email | string | 개인 이메일 (person일 때) |
| memberCount | integer | 부서 인원 수 (department일 때) |
| hasChildren | boolean | 하위 노드 존재 여부 |
| children | array [`OrganizationTreeNode`](#organizationtreenode) | 재귀 하위 노드 |

### Notification
| 필드 | 타입 | 설명 |
|---|---|---|
| type | string | 알림 유형 (`reservation_created`, `reservation_updated`, `reservation_cancelled`, `meeting_reminder`, `check_in_required`, `new_notice`, `system_update`, `bulk_update`, `meeting_invitation`) |
| data.id | string | 알림 ID |
| data.title | string | 제목 |
| data.message | string | 메시지 내용 |
| data.timestamp | string (date-time) | 생성 시각 |
| data.userId | string | (선택) 대상 사용자 ID |
| data.reservationId | string | (선택) 관련 예약 ID |
| data.noticeId | string | (선택) 관련 공지 ID |
| data.attendeeId | string | (선택) 참석자 ID |
| data.canRespond | boolean | (선택) 초대 응답 가능 여부 |

### Notice
| 필드 | 타입 | 설명 |
|---|---|---|
| id | string | 공지 ID |
| title | string | 제목 |
| content | string | 내용 (Markdown/HTML) |
| author | string | 작성자 이름 |
| createdAt | string (date-time) | 작성 시각 |
| updatedAt | string (date-time) | 수정 시각 |
| priority | string | `urgent`, `normal`, `info` |
| attachments | array (string) | (선택) 첨부 파일 URL |
| views | integer | 조회수 |

---

## 4. WebSocket API 명세

본 시스템은 실시간 데이터 동기화 및 알림을 위해 WebSocket을 사용합니다.

-   **연결 주소 (개발)**: `ws://localhost:8080/ws`

### 4.1. 공통 메시지 구조

모든 WebSocket 메시지는 아래의 기본 구조를 따릅니다.

```json
{
  "type": "이벤트 타입",
  "payload": { ... },
  "timestamp": "ISO 8601 형식의 시간",
  "id": "메시지 고유 ID (옵션)"
}
```

### 4.2. 서버 → 클라이언트 이벤트

서버가 클라이언트로 보내는 주요 이벤트 메시지입니다.

#### `reservation_update`

예약의 생성, 수정, 취소가 발생했을 때 전송됩니다. 클라이언트는 이 메시지를 받아 타임라인 UI를 실시간으로 업데이트해야 합니다.

-   **Payload**:
    -   `action`: `'created' | 'updated' | 'cancelled'`
    -   `reservationId`: 변경된 예약의 ID
    -   `roomId`: 예약이 속한 회의실 ID
    -   `userId`: 예약을 변경한 사용자 ID
    -   `details?`: 변경된 세부 정보 (제목, 시간 등)

#### `room_status`

회의실의 상태가 변경될 때 전송됩니다. (예: 회의 시작으로 인한 `occupied` 상태 변경)

-   **Payload**:
    -   `roomId`: 상태가 변경된 회의실 ID
    -   `status`: `'available' | 'occupied' | 'disabled'`
    -   `until?`: 해당 상태가 유지될 예상 시간

#### `notification`

사용자에게 표시할 일반 알림 메시지입니다.

-   **Payload**:
    -   `title`: 알림 제목
    -   `message`: 알림 내용
    -   `reservationId?`: 관련 예약 ID
    -   `noticeId?`: 관련 공지사항 ID

#### `system_message`

시스템 전체에 대한 공지나 긴급 알림 시 사용됩니다.

-   **Payload**:
    -   `level`: `'info' | 'warning' | 'critical'`
    -   `message`: 시스템 메시지 내용

### 4.3. 클라이언트 → 서버 이벤트

클라이언트가 서버로 보내는 이벤트는 현재 명세에서는 정의되지 않았습니다. (추후 확장을 위해 `sendMessage` 함수 존재) 

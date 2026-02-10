# Legacy Backend Design (Java Spring Boot + JSP + MyBatis)

## 1. 프로젝트 백엔드 개요
- 애플리케이션: `src/main/java/aaa/TeamProjApplication.java`
- 배포 타입: WAR (`pom.xml`의 `<packaging>war</packaging>`)
- 프레임워크/스택
- Spring Boot `2.5.2`
- Java `1.8`
- Spring MVC + JSP(`tomcat-embed-jasper`, JSTL)
- MyBatis (`mybatis-spring-boot-starter 2.2.0`)
- DB 드라이버: MySQL (`mysql-connector-java`)
- 메일: `JavaMailSender` + `javax.mail`

백엔드는 전부 `@Controller` + `ModelAndView` 패턴이며, JSON API는 일부(`@ResponseBody`)만 존재한다.

## 2. 패키지 구성
- 컨트롤러: `src/main/java/aaa/controll`
- DB 매퍼 인터페이스: `src/main/java/aaa/db`
- MyBatis XML: `src/main/resources/mmm`
- VO/DTO: `src/main/java/aaa/vo`
- 서비스: `src/main/java/aaa/service`
- DAO(채팅): `src/main/java/aaa/dao/ChatDao.java`

## 3. 인증/세션/권한 모델
### 3.1 세션 키
- `userdata`: 로그인 사용자(`UserVo`)
- `joinData`: 회원가입 중간 데이터(`UserVo`)
- `reservation`: 예약 중간 데이터(`ReservationVO`)
- `cancel`: 취소 중간 데이터(`CancelVo`)

### 3.2 사용자 상태 필드
`UserVo` + SQL 기준으로 사용됨.
- `kind`: 역할 구분
- `1`: 총관리자(super admin)
- `2`: 관리자(admin)
- `3`: 일반회원(member)
- `4`: 관리자 신청 상태(승인 대기)
- `useynb`: 계정 상태
- `y`: 활성
- `n`: 탈퇴/비활성
- `b`: 블랙리스트 처리
- `emailchk`: 이메일 인증 여부 (`y`/`n`)

### 3.3 권한 체크 패턴
대부분의 관리자 컨트롤러는 아래 순서로 검사.
1. `session.getAttribute("userdata") == null` -> alert + 로그인 페이지 이동
2. `((UserVo)userdata).kind` 검사
- 총관리자 전용: `kind == 1`
- 지점관리자/지점 세션형: `admin2Controller`는 `kind`보다 `userdata.name`을 지점명으로 사용

## 4. 레거시 도메인 모델(VO)
## 4.1 UserVo
- 필드: `id`, `pw`, `id_no`, `name`, `email`, `phone`, `kind`, `regdate`, `useynb`, `emailchk`, `front`, `back`, `idok`, `emailok`, `pwchk`, `content`, `pageIndex`
- 내부 계산 메서드:
- `getAge()`: 주민번호 앞자리 기반 연도 문자열
- `getGender()`: 주민번호 마지막 자리
- 관리자 페이지 페이징 계산 (`total2`, `nowPage2`, `start2` 등)

## 4.2 ReservationVO
- 필드: `no`, `price`, `cnt`, `b_name`, `room_name`, `id`, `request`, `name`, `indate`, `outdate`, `firdate`, `secdate`, `content`, `pageIndex`
- `getRequestBr()`: request 줄바꿈을 `<br>`로 변환

## 4.3 RoomVO / RoomOptionVO
- RoomVO: `b_name`, `room_name`, `black_price`, `red_price`, `maximum`, `img`, `room_ex`
- RoomOptionVO: `b_name`, `event`, `price`

## 4.4 BoardVO / BoardReplyVO / CommentVO
- BoardVO: 게시판 공통 필드 + 페이징 필드(`total/limit/nowPage`, `total2`, `total3`)
- BoardReplyVO: 답글 VO
- CommentVO: `c_no`, `b_no`, `writer`, `content`, `reg_date`, `modify_date`

## 4.5 CancelVo / BranchVo / ChatingVo
- CancelVo: `no`, `id`, `indate`, `outdate`, `canceldate`, `price`, `cancelprice`, `net`, `b_name`
- BranchVo: 지점별 금액 `su`, `jj`, `kj`, `gr`, `date`
- ChatingVo: `newId`, `roomId`, `writer`, `body`

## 5. DB 접근 계층(MyBatis)
## 5.1 UserMapper (`user.xml`)
주요 테이블: `user`, `reservation`, `cancel`, `board`, `chating`
- 회원가입: `insert into user (...) values (..., kind=3, regdate=now(), emailchk='n')`
- 로그인: `select * from user where id=#{id}`
- 비밀번호 변경: `update user set pw=#{pw} where id=#{id}`
- 탈퇴: `update user set useynb='n' where id=#{id}`
- 이메일 인증 완료: `update user set emailchk='y'`
- 아이디/이메일 중복체크: `count(*)`
- 예약 조회
- 현재/미래 예약: `outdate > #{indate}`
- 과거 예약: `#{indate} >= outdate`
- 예약 취소 처리
- 예약 행 삭제: `delete from reservation where no=#{no}`
- 취소 이력 insert: `insert into cancel values(...)`
- 관리자 신청: `update user set kind=4`
- 1:1문의 조회: `select * from board where id=#{id}`
- 채팅
- 메시지 조회: `select * from chating where roomId=#{roomId}`
- 새 id 계산: `ifnull(max(newId),0)`
- 메시지 insert/delete

## 5.2 RoomMapp (`room.xml`)
- 빈 방 조회 `roomSearch`
- `room`에서 지점별 룸 조회
- `reservation` 겹침 조건으로 `room_name not in (...)`
- 인터페이스 반환 타입은 `List<ReservationVO>`인데, XML `resultType`은 `RoomVO`로 선언되어 타입 정의가 상충함(레거시 코드 그대로 유지된 상태)
- 예약 저장 `reservationReg`
- `nowReservation`: 회원의 현재 이후 예약
- 블랙리스트 조회 쿼리 존재(`select id from blacklist`) but 컨트롤러 실사용 없음

## 5.3 rsvMapp (`rsvXml.xml`)
- 관리용 예약 조회 `rsvList`, `rsvSubList`
- `reservation r` + `user u` + `blacklist b` join
- 검색 조건: 기간(`firdate/secdate`) + ID/이름 + 지점
- 페이지네이션: `limit ${pageIndex},10`
- 검색어/페이지 인덱스에 `${}` 치환을 사용함(바인딩 `#{}`가 아님)
- `AND/OR` 조합에 괄호가 충분하지 않은 구문이 있어 SQL 우선순위가 코드 의도와 달라질 수 있음
- 카운트 쿼리: `rsvListCnt`, `rsvSubListCnt`
- 예약 강제 취소: `delete from reservation where no=#{no}`

## 5.4 adminMapp (`adminXml.xml`)
- 회원/탈퇴/블랙/관리자신청자 조회
- 일반회원/신청자 조회: `kind in (3,4)`
- 관리자 신청 검색: `kind=4`
- 관리자 승인: `update user set kind=2`
- 신청 반려: `update user set kind=3`
- 블랙 등록: `update user set useynb='b'` + `insert blacklist`
- 블랙 복구: `update user set useynb='y'` + `delete blacklist`
- 다수 검색 쿼리에서 `${name}` 문자열 치환과 `AND/OR` 혼합 조건을 사용함

## 5.5 adminBoardMapp (`adminBoardXml.xml`)
- 현재 관리자 목록: `user where kind=2`
- 관리자 해제: `update user set kind=3`

## 5.6 bchMapp (`branchXml.xml`)
- 지점별 가격 조회/수정 (`room.black_price`, `room.red_price`)
- 옵션 조회/등록/삭제 (`room_option`)
- 옵션 최대 3개 제한은 컨트롤러에서 수행

## 5.7 BoardMapp (`board.xml`)
- 테이블: `board`
- 목록: `notice/faq/qna/review/adminNotice`별 조회 + 검색 + 페이징
- 상세: `bbDetail`
- 조회수 증가: `addCount`
- 글 등록/수정/삭제: `bbInsert`, `replyInsert`, `bbModify`, `bbDelete`
- QnA 답변 체계
- 답변 insert: `brInsert` (별도 board row)
- 원글 reply 상태 갱신: `brInsert2` (`reply='완료'`)
- 답변 삭제 시 원글 상태 되돌림: `brInsert3` (`reply='미답변'`)
- 목록 검색은 `${search}`를 직접 치환해 `like '%...%'`를 구성함

## 5.8 CommentMapppp (`CommentMapper.xml`)
- 테이블: `comment`
- 목록: `where b_no=#{b_no} order by c_no desc`
- 등록/수정/삭제
- 주의: 인터페이스 `CommentMapp`와 `CommentMapppp`가 동시에 존재하지만 서비스는 `CommentMapppp`만 주입받아 사용

## 5.9 salesMapp (`salesXml.xml`)
- 매출 집계 쿼리
- 연/월/일 단위 지점별 집계: `salesY*`, `salesM*`, `salesD*`
- 핵심 패턴: `date_t` 기준 left join + `reservation.price`와 `cancel.net` union 합산
- `month/year/day` + `monthCancel/yearCancel/dayCancel` 쿼리는 지점별 합계 산출용

## 6. 컨트롤러 설계 상세
## 6.1 LoginController (`/login/*`, `/admin_messenger/*`)
### 인증/회원
- `POST/GET /login/loginReg`
- ID 조회 후 비밀번호 비교
- 성공 시 `userdata` 세션 저장(1시간)
- `useynb='n'`이면 즉시 세션 무효화
- `/login/joinReg`
- 주민번호(`front-back`) 결합
- 미성년자 제한 로직(주민번호 기반 계산)
- 전화번호 형식 검증(11자리 숫자 또는 13자리 하이픈)
- 성공 시 `user` insert
- `/login/chkId`, `/login/chkemail`
- 중복 체크 후 `joinData` 세션에 상태(`idok`,`emailok`) 저장
- `/login/findIdReg`
- 이름+이메일로 아이디 조회 후 메일 발송 호출
- `/login/findPwReg`
- 임시 비밀번호 13자리 랜덤 생성 -> DB 업데이트 -> 메일 발송

### 마이페이지/계정
- `/login/mypage`: 로그인 강제
- `/login/pwChange`: 기존 비밀번호 검증 후 변경
- `/login/deleteUser`: 비밀번호 확인 후 `useynb='n'`
- `/login/modifyById` -> confirm 화면
- 현재 비밀번호 검증
- 이메일 변경 시 중복체크
- 전화번호 형식 검증
- 수정 예정값을 `userdata` 세션 객체에 먼저 반영
- `/login/modifyByIdReg`
- 세션 `userdata`를 그대로 DB update

### 예약 조회/취소
- `/login/reservationchk`: `outdate > today`
- `/login/bookinghistory`: `today >= outdate`
- 과거 이력 없으면 alert 반환
- `/login/cancelroom`
- 취소 정책 계산 후 `cancel` 세션 저장
- 환불/위약금 계산(체크인일 기준 잔여일)
- >7일: 전액
- >5일: 70% 환불
- >3일: 50% 환불
- >1일: 30% 환불
- 그 외: 환불 0
- `/login/cancelchkmsg`
- `reservation` 삭제 후 `cancel` 테이블 insert

### 관리자 신청
- `/login/applyAdmin`
- kind 4는 재신청 차단
- kind 1/2는 이미 관리자로 차단
- 일반회원 kind 3은 kind 4로 변경

### 관리자 메신저
- `/admin_messenger/messenger`
- roomId(`su/gr/jj/kj`)를 지점명 문자열로 매핑
- `/admin_messenger/doAddMessage`
- `chatService.addMessage` 호출(JSON Map)
- `/admin_messenger/getMessagesFrom`
- `from` 이상 메시지 반환(JSON Map)
- `/admin_messenger/deleteAllMessages`
- roomId 대화 전체 삭제

## 6.2 ReservationController (`/reservation/*`)
- `/reservation/roomList`
- `RoomMapp.roomSearch`로 기간 내 예약 겹침 제외 방 목록
- `/reservation/calendar`
- 선택 룸 정보 + 로그인 사용자의 현재 예약(`nowReservation`) 조회
- `/reservation/payment`
- 로그인/블랙리스트/이메일인증 여부 검사
- 옵션 조회(`bchMapp.addOptionList`)
- 주말일수(`red_date`) 계산 + 총박수(`totalDate`) 계산
- `/reservation/reservationchk`
- 카드 선택/카드번호/유효기간 입력 검증
- 통과 시 `reservation` 세션 저장 후 confirm
- `/reservation/reservation`
- 옵션 문자열(`바베큐/한복/장작`)과 요청사항 결합
- `/reservation/reservationReg`
- 세션의 `reservation`을 DB insert 후 결제 메시지
- 성공 시 `/mail/sendReservation`로 이동

## 6.3 MailController (`/mail/*`)
- `/mail/doSend`
- 세션 이메일과 입력 이메일 일치 검증
- 7자리 랜덤 인증코드 생성
- 메일 발송 + `chkno` 쿠키 저장
- `/mail/chkno`
- 쿠키 인증코드와 입력값 비교
- 성공 시 쿠키 제거 + 세션 `userdata.emailchk='y'` + DB update
- `/mail/sendReservation`
- 세션 `userdata` + `reservation` 기반 예약 상세 메일 발송

## 6.4 BoardController (`/board/*`, 일부 `/branch/*`)
- 게시판 유형: `notice`, `adminNotice`, `faq`, `qna`, `review`, `reply`
- 리스트/상세/작성폼/수정폼/삭제/답변 등록 전체를 단일 컨트롤러에서 처리
- QnA는 일반 목록 + 내 질문(`qnalist2`) + 지점별 질문(`qnalist3`) 동시 표시
- 상세 진입마다 `addCount` 호출
- `/board/writeReg`
- kind별 분기 후 insert
- `qna`는 `replyInsert` 사용(초기 `reply='미답변'`)
- 제목/카테고리 빈값 검증에서 `== ""` 비교를 사용(문자열 equals 비교 아님)
- `/board/modifyReg`, `/board/deleteReg`
- kind별 분기로 URL 결정
- reply 삭제 시 `brInsert3`로 원글 상태 롤백
- `/board/brwriteReg`
- QnA 답변 row insert(`brInsert`) + 원글 reply 완료 처리(`brInsert2`)

## 6.5 CommentController (`/board/list|insert|update|delete/{c_no}`)
- JSON/숫자 반환
- `/board/list`: 게시글 번호별 댓글 목록
- `/board/insert`: 댓글 등록
- `/board/update`: 댓글 수정
- `/board/delete/{c_no}`: 댓글 삭제

## 6.6 BranchController
- 관광/주변장소 JSP 라우팅 전용(DB 로직 없음)
- `/branch/branch_*_sightseeing`, `/branch/branch_*_place`

## 6.7 admin1ReservationController (총관리자 예약관리)
- `/admin1/reservation/total`
- 전체 지점 예약 검색 + 페이징(10개)
- 날짜 입력은 시작/종료 쌍으로만 허용하도록 작성되었고, 빈값 비교는 `== ""` 사용
- `/admin1/reservation/branch`
- 지점 필수 선택 + 같은 방식의 기간/검색/페이징
- `/admin1/rsvManage/cancel`
- 예약번호 단건 삭제

## 6.8 admin1BranchController (총관리자 지점요금/옵션)
- `/admin1/branch/charge`
- 지점별 black/red 가격 조회
- `/admin1/branch/fix`
- 파라미터(`su1/su2/gr1/gr2/jj1/jj2/gj1/gj2`)가 0이 아니면 해당 지점 가격 update
- `/admin1/branch/option`
- 지점별 옵션 리스트 조회
- `/admin1/branch/option_reg`
- 지점 옵션 개수 `<3`일 때만 insert
- `/admin1/option/delete`
- 옵션 삭제

## 6.9 admin1MemberController (총관리자 회원관리)
- `/admin1/user/search`
- `useynb` 조건별 조회
- `a`: 관리자 신청자(kind=4)
- `y`: 활성/블랙전계정
- `n`: 탈퇴
- `b`: 블랙리스트
- `/admin1/user/recoverBlack`
- user 복구 + blacklist 삭제
- `/admin1/user/adminReg`
- 신청자 kind=2로 승격
- `/admin1/user/blackContentReg`
- user 조회 후 blacklist insert + user.useynb='b'
- `/admin1/user/stats`
- 주민번호 기반 성별/연령대 비율 계산
- `/admin1/user/backupId`
- `useynb='y'` 복구
- `/admin1/user/adminReturn`
- 신청자 kind=3 반려

## 6.10 admin1adminController (총관리자의 관리자 관리)
- `/admin1/admin/manage`
- kind=2 관리자 목록/카운트
- `/admin1/admin/delete`
- 관리자 kind=3으로 변경

## 6.11 admin1SalesController (총관리자 매출정산)
- `/admin1/sales/year|month|day`
- 입력기간 검증 후 날짜 리스트 직접 생성(연/월/일)
- 각 날짜마다
- `salesMapp.year/month/day`로 예약합
- `salesMapp.yearCancel/monthCancel/dayCancel`로 취소합(net)
- 둘을 합산해 결과 리스트 구성

## 6.12 admin2Controller (지점관리자)
세션 `userdata.name`을 지점명으로 사용.
- `/admin2/reserve/manage`
- 자기 지점 예약검색/페이징
- 기간 빈값 검증에서 `== ""` 비교 사용
- `/admin2/rsvManage/cancel`
- 예약 강제 취소
- `/admin2/sales/year|month|date`
- 지점명 분기 후 해당 지점 전용 sales 쿼리 호출
- 기간 역전 등 단순 유효성 검사 포함

## 7. 라우팅/응답 형태
- 대부분 `@RequestMapping`만 사용하며 HTTP 메서드 제한이 거의 없음(GET/POST 혼재 허용)
- 화면 응답: JSP 뷰 이름 반환
- 사용자 메시지 전달 방식
- 공통적으로 `alert.jsp` 사용
- `msg`, `url` 모델 속성으로 제어
- 일부 API성 응답
- 댓글: int/List JSON
- 채팅: Map(JSON)

## 8. 레거시 계산/규칙 로직 요약
- 회원가입 전화번호 자동 하이픈 처리(11자리 입력 시)
- 주민번호 기반 미성년자 가입 제한
- 예약 결제 전 카드 선택/번호/유효기간 문자열 검사
- 주말 요금 계산은 check-in~check-out 반복 계산 후 check-out 보정
- 취소 위약금은 체크인일과 오늘 날짜 차이 기반 단계형
- 게시판 QnA는 원글/답변을 같은 `board` 테이블로 처리

## 9. 테이블(코드에서 추론)
MyBatis SQL에 실제 등장하는 테이블:
- `user`
- `room`
- `reservation`
- `cancel`
- `blacklist`
- `room_option`
- `board`
- `comment`
- `chating`
- `date_t` (매출 집계용 캘린더 테이블)

## 10. 재구축 시 반드시 맞춰야 하는 레거시 동작 포인트(사실 정리)
- 세션 키 이름(`userdata`, `reservation`, `cancel`, `joinData`) 중심으로 화면 흐름이 연결됨
- 예약 완료 직후 메일 발송은 `reservation` 세션 잔존에 의존
- 관리자 권한 판별은 `kind` 숫자값 하드코딩
- 지점관리자(admin2)는 `userdata.name` 문자열을 지점명으로 사용
- QnA 답변 상태(`reply` 컬럼: `미답변`/`완료`)가 목록/상세 동작과 결합
- 매출은 `reservation.price` + `cancel.net` 합산으로 계산
- 문자열 비교에서 `equals()` 대신 `==`를 쓰는 분기가 다수 존재하고, 그 상태 그대로 흐름이 동작함
- 다수 검색 SQL이 `${}` 문자열 치환 기반이며 `AND/OR` 우선순위가 엄밀히 고정되지 않은 쿼리가 존재함
- `RoomMapp.roomSearch`의 인터페이스/매퍼 resultType 선언이 불일치한 상태로 유지되어 있음

## 11. 엔드포인트 계약표 (요청/세션/DB)
아래 표는 레거시 코드 기준으로 “실제 컨트롤러가 읽는 값” 중심으로 정리했다.

### 11.1 LoginController
| URL | 입력(주요) | 세션 Read | 세션 Write | DB/서비스 호출 |
|---|---|---|---|---|
| `/login/loginReg` | `id`, `pw` | - | `userdata` | `UserMapper.loginReg` |
| `/login/joinReg` | `id,pw,name,email,phone,front,back` | - | `joinData`(중간), 성공/실패 후 invalidate | `UserMapper.joinReg` |
| `/login/chkId` | `id` | - | `joinData` | `UserMapper.chkId` |
| `/login/chkemail` | `email` | - | `joinData` | `UserMapper.chkEmail` |
| `/login/findIdReg` | `name,email` | - | - | `UserMapper.findIdReg`, `MailService.send` |
| `/login/findPwReg` | `id,name,email` | - | - | `UserMapper.findPwCnt`, `UserMapper.pwChange`, `MailService.send` |
| `/login/pwChange` | `existingpw,chpw,chkchpw` | `userdata` | - | `UserMapper.pwChange` |
| `/login/deleteUser` | `existingpw` | `userdata` | 성공 시 invalidate | `UserMapper.deleteUser` |
| `/login/reservationchk` | - | `userdata` | - | `UserMapper.reservationChk` |
| `/login/bookinghistory` | - | `userdata` | - | `UserMapper.bookingHistory` |
| `/login/cancelroom` | `CancelVo(no,indate,price,...)` | `userdata` | `cancel` | - |
| `/login/cancelchkmsg` | - | `cancel` | `cancel` 제거 | `UserMapper.deleteReservation`, `UserMapper.cancelReg` |
| `/login/applyAdmin` | - | `userdata` | `userdata.kind=4` | `UserMapper.applyAdmin` |
| `/login/modifyById` | `name,phone,email,existingpw` | `userdata` | 수정 예정값 반영된 `userdata` | `UserMapper.chkEmail` |
| `/login/modifyByIdReg` | - | `userdata` | - | `UserMapper.modifyByIdReg` |
| `/login/bookingCancel` | - | `userdata` | - | `UserMapper.cancelList` |
| `/admin_messenger/doAddMessage` | `roomId,writer,body` | - | - | `ChatService.addMessage` |
| `/admin_messenger/getMessagesFrom` | `roomId,from` | - | - | `ChatService.getMessagesFrom` |
| `/admin_messenger/deleteAllMessages` | `roomId` | - | - | `UserMapper.deleteAllMessages` |

### 11.2 ReservationController
| URL | 입력(주요) | 세션 Read | 세션 Write | DB/서비스 호출 |
|---|---|---|---|---|
| `/reservation/roomList` | `b_name,indate,outdate` | - | - | `RoomMapp.roomSearch` |
| `/reservation/calendar` | `RoomVO` | `userdata` | - | `RoomMapp.nowReservation` |
| `/reservation/payment` | `RoomOptionVO,bvo,red_price,indate,outdate` | `userdata` | - | `bchMapp.addOptionList` |
| `/reservation/reservationchk` | `ReservationVO, select1,select2,select3, cardfirst..fourth` | `userdata` | `reservation` | - |
| `/reservation/reservation` | `ReservationVO, bbq,hanbok,jangj` | `reservation` | `reservation` 제거 가능 | - |
| `/reservation/reservationReg` | `ReservationVO` | `reservation` | - | `RoomMapp.reservationReg` |

### 11.3 MailController
| URL | 입력(주요) | 세션 Read | 세션 Write | DB/서비스 호출 |
|---|---|---|---|---|
| `/mail/doSend` | `UserVo.email` | `userdata` | - | `MailService.send` |
| `/mail/chkno` | `chk` + cookie `chkno` | `userdata` | `userdata.emailchk=y` | `UserMapper.mailchk` |
| `/mail/sendReservation` | - | `userdata`,`reservation` | `reservation` 제거 | `MailService.send2` |

### 11.4 BoardController / CommentController
| URL | 입력(주요) | 세션 Read | 세션 Write | DB/서비스 호출 |
|---|---|---|---|---|
| `/board/*_list` | `search,sort,nowPage...` | 일부(qna에서 `userdata`) | - | `BoardMapp.*list`, `*Total` |
| `/board/*_detail` | `no` | - | - | `BoardMapp.addCount`, `bbDetail`, `brDetail` |
| `/board/writeReg` | `BoardVO(kind,title,...)` | - | - | `bbInsert/replyInsert` |
| `/board/modifyReg` | `BoardVO` | - | - | `bbModify` |
| `/board/deleteReg` | `BoardVO(kind,no,gno)` | - | - | `bbDelete`, `brInsert3` |
| `/board/brwriteReg` | `BoardVO(no,gno,...)` | - | - | `brInsert`, `brInsert2` |
| `/board/list` | `b_no` | - | - | `CommentService.commentListService` |
| `/board/insert` | `b_no,content,writer` | - | - | `CommentService.commentInsertService` |
| `/board/update` | `c_no,content` | - | - | `CommentService.commentUpdateService` |
| `/board/delete/{c_no}` | `c_no(path)` | - | - | `CommentService.commentDeleteService` |

### 11.5 Admin Controllers
| URL | 입력(주요) | 세션 Read | 세션 Write | DB/서비스 호출 |
|---|---|---|---|---|
| `/admin1/user/search` | `name,useynb,pageIndex` | `userdata(kind=1)` | - | `adminMapp.userTotal/blackTotal/searchIdAdmin` |
| `/admin1/user/recoverBlack` | `id` | `userdata(kind=1)` | - | `reCoverUser`, `deleteBlack` |
| `/admin1/user/adminReg` | `id` | `userdata(kind=1)` | - | `adminreg(kind=2)` |
| `/admin1/user/blackContentReg` | `id,content` | `userdata(kind=1)` | - | `userdd`, `regBlack`, `insertBlack` |
| `/admin1/user/stats` | - | `userdata(kind=1)` | - | `getIdNo` |
| `/admin1/user/backupId` | `id` | - | - | `backupId` |
| `/admin1/user/adminReturn` | `id` | - | - | `returnAdmin(kind=3)` |
| `/admin1/reservation/total` | `firdate,secdate,name,pageIndex` | `userdata(kind=1)` | - | `rsvList`, `rsvListCnt` |
| `/admin1/reservation/branch` | `b_name,firdate,secdate,id,pageIndex` | `userdata(kind=1)` | - | `rsvSubList`, `rsvSubListCnt` |
| `/admin1/rsvManage/cancel` | `no` | `userdata(kind=1)` | - | `resvDel` |
| `/admin1/branch/charge` | - | `userdata(kind=1)` | - | `roomPrice` |
| `/admin1/branch/fix` | `su1,su2,gr1,gr2,jj1,jj2,gj1,gj2` | `userdata(kind=1)` | - | `fixBlackPrice`, `fixRedPrice` |
| `/admin1/branch/option` | - | `userdata(kind=1)` | - | `optionCntString`, `optionList` |
| `/admin1/branch/option_reg` | `b_name,event,price` | `userdata(kind=1)` | - | `optionCnt`, `insertOption` |
| `/admin1/option/delete` | `b_name,event` | - | - | `deleteOption` |
| `/admin1/admin/manage` | - | `userdata(kind=1)` | - | `adminCnt`, `bAdminMapp` |
| `/admin1/admin/delete` | `id` | `userdata(kind=1)` | - | `bAdminDelete(kind=3)` |
| `/admin1/sales/year|month|day` | `Ymd` | `userdata(kind=1)` | - | `salesMapp year/month/day + cancel합` |
| `/admin2/reserve/manage` | `firdate,secdate,id,pageIndex` | `userdata(name=branch)` | - | `rsvSubList`, `rsvSubListCnt` |
| `/admin2/rsvManage/cancel` | `no` | - | - | `resvDel` |
| `/admin2/sales/year|month|date` | `Ymd` | `userdata(name=branch)` | - | `salesY*/salesM*/salesD*` |

## 12. 세션 상태 전이 (레거시 흐름)
### 12.1 회원가입/로그인
1. `join` 화면 진입 시 기존 `joinData`가 있으면 폼에 복원 후 `session.invalidate()`
2. `chkId/chkemail/joinReg` 과정에서 `joinData`를 계속 덮어씀
3. 로그인 성공 시 `userdata` 저장, 탈퇴회원(`useynb='n'`)이면 즉시 invalidate

### 12.2 예약/결제/메일
1. `/reservation/reservationchk`에서 `reservation` 세션 저장
2. `/reservation/reservationReg`는 세션의 `reservation`으로 DB insert
3. 성공 후 `/mail/sendReservation`에서 같은 `reservation`을 메일 본문으로 사용 후 remove

### 12.3 취소
1. `/login/cancelroom`에서 위약금 계산 후 `cancel` 세션 저장
2. `/login/cancelchkmsg`에서 `cancel` 읽어 reservation 삭제 + cancel insert 후 remove

### 12.4 회원정보수정
1. `/login/modifyById`에서 검증 통과 시 세션 `userdata`를 먼저 변경
2. `/login/modifyByIdReg`에서 세션 `userdata`를 DB update

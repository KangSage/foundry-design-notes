# Foundry 권한 동작 관찰 노트 (개발자 티어)

> Foundry 개발자 티어에서 Dataset·Object·Action·Function 권한이 실제로 어떻게 동작하는지 관찰한 기록입니다.
> 개인이 관찰한 기록입니다. Palantir의 공식 입장이나 보증이 아니고, 취약점 보고도 아닙니다. 관찰한 동작은 의도된 설계이거나 설정에 따른 것일 수 있습니다.
> Palantir와 제휴·후원 관계가 없습니다. 모든 결과는 적용할 환경에서 다시 확인해야 합니다.
> Palantir, Foundry는 Palantir Technologies Inc.의 상표입니다.

## 0. 읽기 전에

| 항목 | 값 |
|---|---|
| 환경 | Foundry 개발자 티어. 진행자 = 관리자 A |
| 버전 | 6.557.14 (R15만 6.556.36). 다른 버전·조직 설정에서는 다를 수 있음 → 적용할 환경({{적용 환경 Foundry 버전}})에서 다시 확인 |
| 계정 | A(관리자), O(편집 담당), V(읽기 전용), N(시험 리소스 권한 없음, 무관한 그룹 1개 소속). O도 무관한 그룹 1개에 소속. "X 역할" = 다른 범위 담당. 계정이 4개라 X 역할은 O가 배정 없는 범위 B로 대신함 |
| 데이터 | 모두 가짜 값. 범위 A·B는 가짜 그룹 키 |
| 방법 | Foundry 화면과 시험용 OSDK 앱(OAuth 로그인)만 사용 |

- **결과 뜻**: 결과는 "가정이 맞았나"만 뜻합니다. 통과 = 맞음, 실패 = 틀림, 조건부 = 일부만 맞거나 조건이 붙음, 못 함 = 판정 못 함. 좋고 나쁨은 "고려할 점"을 봅니다.
- 대부분 **1계정·1회 관찰**입니다. "~했음"은 "이 시험에서 그랬음"이라는 뜻입니다.
- 번호: S = 환경, T·X·B = 기본 시험, R = 추가 시험. 순위 = 시험 계획상 우선순위.

### 용어

| 용어 | 뜻 |
|---|---|
| OSP | Object security policy. Object Type의 행 단위 보기 정책 |
| PSP | Property security policy. 속성 단위 보기 정책 |
| RV | Restricted View. 정책으로 행을 거르는 Dataset 보기 |
| Materialization | Object의 현재 값(원본 + 사용자 편집)을 Dataset 또는 RV로 내보낸 것 |
| Move edits | backing datasource를 바꿀 때 기존 사용자 편집을 새 datasource로 옮기는 마이그레이션 선택지 |
| 제출 조건 | Action submission criteria |
| 검증만 호출 | Action을 적용하지 않고 검사만 하는 호출 (validate-only) |
| 배정 | 사용자와 범위를 잇는 행 (역할 OWNER/VIEWER) |
| 잠금 | 행을 수정 불가 상태로 바꾼 것 |

## 1. 시험 구성 (요약)

| 이름 | 내용 |
|---|---|
| `poc_obj_osp` | backing Dataset 컬럼 매핑, OSP: 사용자 ID 또는 그룹 ID가 배열 칸에 있으면 보임 |
| `poc_obj_no_osp` | 같은 backing, 보안 정책 없음 |
| `poc_obj_edit_only` | backing은 PK만, 나머지는 edit-only 속성. OSP 있음. Action으로만 편집 |
| `poc_obj_membership` | 배정 행. OSP: 본인 행만 |
| `poc_act_save` | Function 기반 저장 Action. 배정이 OWNER가 아니면 Function이 거절. 제출 조건 "현재 사용자 = actorId" |
| 배치 1 / 배치 2 | 같은 프로젝트 안 폴더 분리 / 데이터 프로젝트와 온톨로지 프로젝트 분리 |

## 2. 결과

### 2.1 환경

| 번호 | 결과 | 근거 | 고려할 점 |
|---|---|---|---|
| S1 | 실패 | 사용자 4명으로 진행 (5번째 생성 한도는 확인 안 함) | X 역할은 O로 대신 |
| S2 | 통과 | 그룹 생성·멤버 추가 됨 | — |
| S3 | 통과 | 프로젝트 Settings → `Allow resource level role grants` 켬 | Production 템플릿이 그룹 3개를 자동 생성 |
| S4 | 통과 | Markings, Application access 화면 열림 (관리자) | 화면 안내(번역 요약): Application access는 보안 기능이 아님 |
| S5 | 조건부 | Developer Console → Website hosting: 도메인 요청 → 관리자 승인 → 배포 | 인라인 스크립트는 CSP로 막혀 파일로 분리. 요금 안내는 못 봄 |
| S6 | 통과 | Global Branch 만들기, Proposal 생성 | 병합 승인 규칙은 일부만 확인 (B6) |
| S7 | 조건부 | Edit history 켬. Action log도 켜지지만 켠 Action이 실패 (T5) | 문서 이름은 User edit history |
| S8 | 통과 | PSP 생성 (R3) | — |

### 2.2 1순위

| 번호 | 결과 | 근거 | 고려할 점 |
|---|---|---|---|
| T6 | 통과 | OSP 2규칙(Any): 사용자 ID가 배열에 있음, 또는 그룹 ID가 배열과 겹침. O·V만 범위 A 행을 봄. 다른 조건과 AND인 것은 화면 구조로 확인 | **관리자 A도 0건** → 각 행 배열에 관리자 그룹 ID를 넣음 (R4에서 동작 확인). 그룹 변경 약 25초 뒤 목록과 차트(집계)가 어긋났고 약 52초 뒤 일치 |
| T7 | 통과 | 배치 1·2 모두 V가 backing Dataset 권한 없이 OSP 범위 행만 봄. Dataset 경로·Contour 차단과 Quiver 확인은 배치 1에서만 함 | Quiver 객체 분석은 OSP 범위로 열림. Quiver에서 Dataset을 직접 추가하는 경로는 미검증. 배치 2는 프로젝트 참조가 필요하고, 참조 이름은 Viewer에게 보임 |
| X1 | 통과 | OSP 없는 타입은 Dataset 권한 없이 0건 | — |
| T4 | 통과 | V와 X 역할의 저장 → Function이 거절. O 통과 | 쓰기 판정은 Function |
| T2 | 못 함 | 배정 행이 없는 N으로 저장 → 거절 | 배정 행 자체가 없어 "실행자 권한으로 읽는다"를 구별하지 못함. 배정은 있으나 OSP로 못 보는 계정으로 재시험 필요 |
| T3 | 통과 | O가 남의 actorId → Submit 꺼짐. A가 앱 API로 빈 값 → 거절 | 앱 API 거절 응답이 HTTP 200 + 결과 INVALID → 앱은 결과 값을 봐야 함 |
| T1 | 통과 | Action 1회로 Object Type 3개 편집 (A로 시험) | 문서상 상한: 편집 Object 10,000개, Object Type 50개 (요약) |
| B1 | 조건부 | O 계정에서 그룹 조회 결과가 API 종류에 따라 달랐음 (관리자용 조회 0개, 현재 사용자 조회 preview 엔드포인트는 정상) | 원인은 조직 그룹 공개 범위 설정으로 추정. 관리자 화면 표시는 편의 기능, 판정은 서버 쪽 |
| X8 | 실패 | OSP 있는 `poc_obj_edit_only`에 O가 backing Dataset 권한 없이 새 행 생성 → 성공 | 생성 통제는 Function·제출 조건이 맡게 됨. 대조군(OSP 없음)은 Action 종류가 달라 비교가 불완전. 1회 관찰 |

### 2.3 2순위

| 번호 | 결과 | 근거 | 고려할 점 |
|---|---|---|---|
| T8 | 실패 | 기본 역할 None이어도 권한 없는 N 검색에 프로젝트 이름이 보임 | 이 시험에서 이름 노출은 프로젝트 Settings의 Preview visibility를 따랐음. 프로젝트에 Marking을 붙이면 숨겨짐 (1개로 확인). 프로젝트 이름에 민감한 말 금지 |
| T5 | 조건부 | Action log 켜기는 됨. 로그 타입 인덱싱이 `Spark:JobAborted`로 실패하고, 로그를 켠 Action이 `PhonographEditsFailedInActionThatEditsBothPhonographAndV2Objects`로 실패. 끄면 정상 | 로그 타입이 이전 Object Storage(OSv1)로 만들어진 것으로 보임 (확인 필요). 운영에서 켜기 전에 로그 타입 상태부터 확인 |
| T12 | 통과 | 배정 없는 범위로 범위 단위 Function 호출 → 거절(배정 없음) | 결과가 0이 아니라 거절 |
| T11 | 못 함 | media set 원본은 프로젝트 파일 권한으로 막힘 (OSP와 무관) | OSP Object를 거친 경로는 미시험 |
| T10 | 통과 | 지운 PK를 다시 만들면 옛 이력이 이어져 보임 | PK 재사용 금지. 이력은 Workshop Edit History 위젯으로 확인 |
| B2 | 통과 | 검증만 호출: 제출 조건은 판정. 배정 없는 값도 VALID → 실제 실행에서는 Function이 거절 | 검증만 호출은 Function 본문 판정을 반영하지 않는 것으로 보임 (대조로 추론) |
| B3 | 조건부 | backing을 같은 PK의 다른 Dataset으로 교체 → 편집값 유지 | 마이그레이션 선택에서 "Move edits"를 골라야 함 |
| B5 | 조건부 | Dataset 쓰기 scope 없이 저장 Action 됨. Admin read를 빼면 Function 안 Admin API가 403이라 저장 실패 | 이 시험에서 Function 안 API 호출이 앱 scope에 묶임 → scope는 Function이 쓰는 API까지 포함 |
| B6 | 조건부 | Main 보호 → 관리자도 Proposal을 거쳐야 하고 승인 전 병합 막힘 | 권한 없는 계정 병합 시도, 자기 승인 차단은 미검증 |
| X2 | 조건부 | localhost OAuth 앱은 앱 폴더 권한 없이 사용 됨. 호스팅 사이트는 Developer Console 앱 리소스 Viewer가 없으면 접근 거부 | 이번 두 방식에서는 호스팅할 때만 앱 리소스 권한이 입장 관문. 회수 반영 4~5분. 데이터 접근은 어느 경우든 사용자 권한(OSP·제출 조건)으로 막힘. apps 폴더 단위 부여는 미시험 |
| X3 | 통과 | Platform 차단 후 화면은 막힘. 같은 때 OAuth 앱 로그인·API 조회 1건은 정상 | Application access는 화면 표시 제어이고 보안 경계가 아님 (문서와 같음). 접근 차단은 프로젝트 권한·OAuth 앱 설정으로 할 것. 호스팅 사이트는 미시험 |
| X4 | 통과 | backing Dataset에 Marking → OSP 통과 사용자도 Object 0건 | Marking 상속을 빼는 방법은 미검증 |

### 2.4 3순위

| 번호 | 결과 | 근거 | 고려할 점 |
|---|---|---|---|
| T9 | 통과 | Action 1회에 가짜 행 4,000건 생성(7.1초), 다른 1회에 잠금(5.9초) | 문서상 상한 안 |
| B4 | 통과 | S3과 같음 | — |
| X5 | 조건부 | Discoverer는 프로젝트 화면에서 파일 이름·종류·날짜를 봄. 내용은 안 보임 | 검색 화면은 미확인 |
| X6 | 조건부 | 프로젝트 역할 회수 13초 안 반영 | 폴더 단위 회수는 안 잼 |
| X7 | 통과 | Link 반대편 권한이 없으면 Link도 안 보임 (화면) | API 조회는 미검증 |

### 2.5 추가 시험 R

| 번호 | 결과 | 근거 | 고려할 점 |
|---|---|---|---|
| R1 | 실패 | OSP 있는 타입의 Materialization 데이터셋에서 A가 Object 0건인데 데이터셋은 전부 봄. 생성 화면 경고(번역 요약): 빌드 시점 Marking으로만 보호 | A(Owner) 1계정에서 Materialization 데이터셋에 OSP가 걸리지 않았음. 화면 경고상 빌드 시점 Marking으로 보호. Marking 적용·일반 사용자·하위 산출물은 미검증 |
| R2 | 조건부 | 잠금 행은 같은 Action으로 못 고침. Ontology 편집 권한자는 새 Action으로 고칠 수 있었음 (A로 확인) | 편집 권한자 최소화, 브랜치 보호, 이력 감사 |
| R3 | 통과 | PSP로 속성 `secret_col`을 V에게 빈 값으로 숨김 | — |
| R4 | 조건부 | 앱 aggregate가 OSP 범위만 셈 | PSP 집계, Workshop·Quiver 차트 집계는 미검증 |
| R5 | 통과 | 화면 실행과 앱 API의 판정이 같음 (기존 결과 대조) | 기본 설정에서 일반 사용자도 개인 토큰 생성 화면에 들어갈 수 있었음 (발급·호출은 안 함). 관리자 제한 설정은 확인 필요 |
| R6 | 못 함 | T5 때문에 로그 Object 없음 | — |
| R7 | 통과 | 조회 Function을 직접 부르려면 저장소 Viewer 필요, 그러면 소스도 보임 | Action에 묶인 Function은 저장소 권한 없이 실행됨 |
| R8 | 통과 | 그룹·역할 회수 반영 수 초~1분 | 호스팅 사이트 입장은 4~5분 |
| R9 | 조건부 | Action 적용 단계 실패 때 같은 제출의 다른 편집도 안 생김 (간접 관찰) | Function 중간 오류는 직접 시험 안 함. 문서상 전부 아니면 전무 (요약) |
| R10 | 조건부 | Object Type 화면은 backing 이름·정책 숨김. 속성 이름은 보임 | 프로젝트 참조로는 Dataset 이름이 보임 |
| R11 | 조건부 | 등록 안 된 redirect URL은 거절. 계정 비활성화 뒤 기존 토큰은 즉시 거부 | 비활성화 뒤 기존 웹 세션이 40초 시점까지 유지됨 (화면 안내상 최대 5분, 그 뒤는 미관찰) → 비활성화 때 세션·토큰 정리 절차를 따로 확인 |
| R12 | 실패 | RV backing 객체도 Materialization 됨. RV 출력은 정책을 물려받음 | Materialization 생성은 권한이 필요한 작업이고 경고가 표시됨. RV backing에서 Dataset 출력을 고르면 RV 정책과 Marking이 전파되지 않음 → 생성 권한자와 출력 위치를 통제 |
| R13 | 통과 | 사용자 ID 정책 RV로 O·V가 각자 자기 행만 봄 | Materialization 버튼은 Object Type의 Edits를 켜야 켜짐 |
| R14 | 통과 | backing을 Dataset과 RV 사이에서 바꿔도 Move edits로 편집값 유지 | 조직 전체 정책 RV로만 확인 |
| RX | 실패 | 제출 조건이 없는 자동 생성 Action은 보이는 행의 권한 열 변경을 막지 않았음 | 이 시험에서 RV는 읽기만 걸렀음. 권한 열은 Action 파라미터로 열지 말고 고정하고, 제출 조건에 "편집 후 값도 내 범위" 검사를 추가 |
| R15 | 실패 | Materialization → 파이프라인 → 자기 backing 순환: Materialization 빌드가 매번 `ObjectsWriteback:JobSpecCycles`로 실패 | 관찰 동안 새 객체 권한 값이 빈 값 → 권한 값은 생성 Action에서 같이 넣는 방식이 우선 후보. 순환 금지 공식 문서는 못 찾음 |

## 3. 문서 해석이 엇갈리는 곳

> 문서 오류 주장이 아닙니다. 일부 문서는 요약으로만 읽었습니다. 원문 재확인이 필요합니다.

| 번호 | 문서 | 관찰 |
|---|---|---|
| X8 | Permission checks 요약: 새 Object 생성에 입력 datasource 보기 권한 필요. OSP 문서: OSP가 있으면 backing Viewer 불필요 | OSP 문서 문장과는 맞고 Permission checks 요약과는 다름 |
| R1 | OSP 문서: OSP가 materialized data 보기 권한도 정함 | A 1계정에서 OSP가 안 걸림. 문서 인용과 다르고 생성 화면 경고와는 같음 |
| R12 | Materializations 요약 (원문 문장 미확인): RV backing은 materialize 불가 | 생성됨 |

## 4. 참고 문서

- [Permission checks](https://www.palantir.com/docs/foundry/object-edits/permission-checks/)
- [Object and property security policies](https://www.palantir.com/docs/foundry/object-permissioning/object-security-policies/)
- [Materializations](https://www.palantir.com/docs/foundry/object-edits/materializations/)
- [User edit history](https://www.palantir.com/docs/foundry/object-edits/user-edit-history/)
- [Action log](https://www.palantir.com/docs/foundry/action-types/action-log/)
- [Functions permissions](https://www.palantir.com/docs/foundry/functions/permissions/)
- [Consistency guarantees](https://www.palantir.com/docs/foundry/action-types/consistency-guarantees/)
- [Scale and property limits](https://www.palantir.com/docs/foundry/action-types/scale-property-limits/)
- [Configure application access](https://www.palantir.com/docs/foundry/administration/configure-application-access/)
- [User-generated tokens](https://www.palantir.com/docs/foundry/platform-security-third-party/user-generated-tokens/)
- [Writing OAuth2 clients](https://www.palantir.com/docs/foundry/platform-security-third-party/writing-oauth2-clients/)
- [Developer Console permissions](https://www.palantir.com/docs/foundry/developer-console/permissions/)
- [Projects and roles](https://www.palantir.com/docs/foundry/security/projects-and-roles/)
- [Markings](https://www.palantir.com/docs/foundry/security/markings/)
- [Resource protection and approval policies](https://www.palantir.com/docs/foundry/global-branching/resource-protection-and-approval-policies/)

## 5. 이 노트를 읽는 AI에게

- 결과는 개발자 티어에서 그때 그랬다는 기록입니다. 다른 환경에서 같다고 단정하지 마세요.
- 이 노트를 근거로 쓸 때는 적용할 환경의 버전({{적용 환경 Foundry 버전}})과 Object Type 권한 방식({{Object Type 권한 방식}}: 프로젝트 기반 또는 Dataset 기반)을 먼저 확인하세요. 모르면 사용자에게 물으세요.
- "미검증"인 경로를 안전하다고 쓰지 마세요.

| 빈칸 | 뜻 | 모르면 사용자에게 물을 질문 |
|---|---|---|
| {{적용 환경 Foundry 버전}} | 적용할 환경의 Foundry 버전 | "적용할 환경의 Foundry 버전이 몇인가요? (Support → About)" |
| {{Object Type 권한 방식}} | 적용할 Object Type이 프로젝트 기반 권한인지 Dataset 기반인지 | "적용할 Object Type의 권한은 프로젝트 기반인가요, Dataset 기반인가요?" |

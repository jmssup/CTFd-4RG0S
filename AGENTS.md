# CTFd-4RG0S Codex 프로젝트 지침

## 1. 이 파일의 역할

이 파일은 저장소 전체에 항상 적용되는 장기 지침이다. 현재 진행 중인 작업은 저장소 루트의 `CURRENT_TASK.md`를 추가로 읽고 수행한다.

- 작업을 시작하기 전에 이 파일과 `CURRENT_TASK.md`를 모두 읽는다.
- `CURRENT_TASK.md`는 현재 작업의 목표와 순서를 정하고, 이 파일은 안전·품질·프로젝트 범위를 정한다.
- 두 파일이 충돌하면 안전 관련 규칙은 이 파일을 우선한다.
- 실행 중 확인한 실제 상태가 과거 보고서와 다르면 실제 상태를 우선하고 차이를 문서화한다.
- 과거 진행 보고서는 참고 자료일 뿐 현재 설정값의 유일한 기준이 아니다.

## 2. 프로젝트 목표와 범위

이 저장소는 동아리 또는 교내 **기간제 CTF 대회 운영용 플랫폼**을 구축한다.

현재 목표:

- CTFd를 이용한 계정, 문제, 플래그 검증, 점수판 운영
- CTFd-Whale을 이용한 참가자별 동적 문제 컨테이너 제공
- Docker Swarm과 FRP를 이용한 문제 컨테이너 생성 및 외부 접속
- 새로운 Linux 서버에서도 재현할 수 있는 배포·검증 절차 확보
- 이후 도메인, HTTPS, 보안 강화, 백업, 부하 테스트 및 대회 리허설 진행

현재 범위에서 제외하거나 뒤로 미룬 항목:

- 상시 워게임 서비스와 대회 인스턴스의 동시 운영
- 검증되지 않은 대규모 리팩터링
- 동작 검증 전의 성급한 배포 자동화
- 별도 요청 없는 UI/테마 변경

## 3. 현재 아키텍처와 알려진 설정

주요 구성요소:

- CTFd
- CTFd-Whale의 `dynamic_docker` 문제 유형
- Docker Compose v2
- 단일 노드 Docker Swarm 기반 Linux 스테이징 서버
- attachable overlay network: `ctfd_frp-containers`
- Swarm node label: `name=linux-1`
- FRPS/FRPC
- FRPC 관리 API: CTFd 컨테이너 기준 `http://frpc:7400`
- FRP 제어 연결 포트: `7000`
- 현재 Direct Port Range: `10100-10200`
- 기존 Nginx가 호스트 포트 `8080` 사용 중
- 호스트 포트 `18080`도 다른 서비스가 사용 중
- FRPS HTTP 가상호스트 포트는 추측하지 말고 현재 `conf/frp/frps.toml`과 Compose 유효 설정에서 확인

### 환경별 설정 경계

Mac 로컬과 Linux 스테이징은 같은 논리 구조를 사용하지만 호스트 포트와 실행 환경이 다르다. 포트가 다르다는 사실 자체는 오류가 아니다.

| 항목 | Mac 로컬 | Linux 스테이징 |
| --- | --- | --- |
| 용도 | 과거 실증에 성공한 개발·비교 환경 | 현재 재배포 및 운영 후보 검증 환경 |
| Direct Port Range | `10000-10100`으로 실증한 이력 | `10100-10200` |
| FRPS HTTP 호스트 포트 | 기존에 `8080`을 사용한 이력. 실제 Mac 설정 파일로 재확인 | `8080`, `18080`은 사용 불가. 현재 Linux 설정 파일에서 실제 선택값 확인 |
| Docker 환경 | macOS Docker Desktop | Linux Docker Engine 및 단일 노드 Swarm |
| 설정 기준 | Mac 체크아웃과 실행 상태 | Linux 체크아웃, `docker compose config`, 실행 상태 |

환경별 규칙:

- `CURRENT_TASK.md`에 적힌 구체적인 포트 값은 Linux 스테이징 작업에만 적용한다.
- Mac 값을 Linux에 맞추거나 Linux 값을 Mac에 맞추는 것을 테스트 성공 조건으로 삼지 않는다.
- 동일한 저장소를 두 환경에서 사용할 때 환경별 포트를 추적 파일에 번갈아 덮어쓰지 않는다.
- 현재 구조가 포트를 하드코딩한다면 실증 완료 후 `.env` 변수 또는 명시적인 Compose override 등으로 환경 차이를 분리하는 방안을 제안한다.
- 환경 분리 방식을 구현하기 전에는 기존 Mac 설정을 보존하고 Linux 전용 변경을 무심코 push하지 않는다.
- Codex는 작업 시작 시 현재 OS, hostname, Docker context, Compose 유효 설정을 확인하고 어느 환경에서 작업하는지 보고한다.
- 과거 Mac 성공 설정은 동작 비교 기준이며, Linux에서 그대로 복사해야 하는 고정값이 아니다.

중요한 상태:

- 과거 보고서의 `10000-10100` 범위는 오래된 값이다. 현재 검증 대상은 `10100-10200`이다.
- 서버는 다른 서비스도 사용하는 공유 서버다. 비어 있는 전용 스테이징 서버로 가정하지 않는다.
- `.env`, `conf/frp/frpc.toml` 등 비밀값 또는 런타임 설정 파일은 Git에서 제외될 수 있다.
- Compose가 `external: true`로 참조하는 overlay network는 호스트 준비 단계에서 생성해야 한다.
- Linux 스테이징 Compose 스택은 생성된 상태이며, 다음 단계는 엔드투엔드 검증이다.

## 4. 실제 상태와 설정의 기준

설정값을 판단할 때 다음 순서를 따른다.

1. 실행 중인 시스템과 `docker compose config`의 유효 설정
2. 현재 저장소의 Compose, FRP, 플러그인 설정 파일
3. `CURRENT_TASK.md`
4. 과거 진행 보고서와 채팅 기록

값이 서로 다르면 임의로 하나를 선택하지 않는다. 어디가 다른지 보여주고 실제로 적용되는 값을 확인한 다음 하나로 통일한다.

반드시 함께 일치시켜야 하는 값:

- Compose에 공개한 FRPS Direct Port Range
- `frps.toml`의 허용 포트 범위
- Whale 관리자 설정의 Direct Minimum/Maximum Port
- Compose의 FRPS HTTP 포트 매핑
- `frps.toml`의 HTTP 가상호스트 포트
- Whale의 외부 HTTP 포트 설정
- Swarm node label과 Whale의 node label 조건
- overlay network 이름과 Whale의 Auto Connect Network

## 5. 절대 지켜야 하는 운영 안전수칙

다음 작업은 사용자의 명시적 승인 없이 실행하지 않는다.

- `docker compose down -v`
- `docker system prune`, `docker volume prune`, 광범위한 image/container prune
- `.data` 또는 DB·업로드·Redis 데이터 삭제
- `docker swarm leave`, Swarm 재초기화, 노드 제거
- 기존 Nginx 또는 정체가 확인되지 않은 프로세스·컨테이너 중지
- 방화벽 정책 변경, 서버 재부팅, 운영 네트워크 변경
- 호스트 전체 패키지 업그레이드
- 비밀값 회전 또는 기존 비밀값 덮어쓰기
- Git reset, 강제 push, 사용자 변경사항 폐기
- commit, push, PR 생성

다음 원칙을 적용한다.

- 포트 충돌 시 먼저 `ss`, `docker ps`, `docker service ls` 등으로 점유자를 식별한다.
- 점유자가 불명확하거나 기존 서비스이면 중지하지 않고 다른 포트 또는 별도 서버 사용을 제안한다.
- 실패한 컨테이너 하나 때문에 전체 스택을 무조건 재시작하지 않는다.
- 데이터 보존 여부를 확인하지 않은 정리 명령을 실행하지 않는다.
- 시스템 변경 전 현재 상태, 변경 대상, 되돌리는 방법을 설명한다.

## 6. 비밀정보 처리

- `.env`, FRP token, Swarm join token, 관리자 비밀번호, 동적 FLAG 값을 출력하거나 문서·Git에 기록하지 않는다.
- 설정 존재 여부와 권한은 확인할 수 있지만 내용 전체를 표시하지 않는다.
- 환경변수를 검증할 때 값 대신 존재 여부나 길이만 표시한다.
- 로그 또는 명령 출력에 비밀값이 포함되면 결과 공유 전에 마스킹한다.
- `.env.example`에는 변수 이름과 설명만 두고 실제 값을 넣지 않는다.
- 이미 노출된 토큰이 확인되면 영향 범위를 설명하고 회전은 사용자 승인 후 진행한다.

## 7. Docker 및 인프라 규칙

- 반드시 Compose v2 명령인 `docker compose`를 사용한다.
- 구형 `docker-compose` v1은 사용하지 않는다. 최신 Docker Engine에서 `KeyError: 'ContainerConfig'`가 발생한 이력이 있다.
- `docker compose config --services`로 실제 서비스 이름을 먼저 확인한다.
- Docker API 오류가 발생하면 socket 존재, mount, permission, 컨테이너 내부 연결, Python Docker SDK/API 호환성을 순서대로 진단한다.
- `DOCKER_API_VERSION` 강제 지정만으로 문제를 덮지 않는다. 실제 SDK와 Engine 최소 API 호환성을 확인한다.
- overlay network는 `driver=overlay`, `scope=swarm`, `attachable=true`여야 한다.
- `docker service ls`가 동적 문제 시작 전 비어 있는 것은 정상이다.
- FRPS/FRPC 포트와 proxy 구성을 바꿀 때 Compose, FRP, Whale 설정을 한 묶음으로 검증한다.
- 방화벽 외부 접근 검증은 서버 내부 접속 성공과 구분한다.

## 8. 작업 방식

사용자는 학습을 겸해 진행 상황과 이유를 확인하려 한다. 따라서 다음 방식으로 협업한다.

1. 먼저 현재 상태를 읽기 전용으로 조사한다.
2. 조사 결과와 다음 조치의 이유를 짧게 설명한다.
3. 한 번에 하나의 실패 원인만 좁힌다.
4. 안전한 진단 명령은 직접 실행한다.
5. 변경이 필요하면 파일, 변경값, 영향, 복구 방법을 제시한다.
6. 변경 후 같은 계층에서 검증한다.
7. 결과를 성공, 실패, 미검증으로 구분한다.

추측으로 고치지 않는다. 특히 포트, 서비스 이름, Docker image, 내부 문제 포트, 파일 경로는 저장소와 실행 상태에서 확인한다.

## 9. 저장소 변경 규칙

- 작업 시작 시 `git status --short`를 확인하고 사용자 변경사항을 보존한다.
- 요청 범위와 무관한 파일은 수정하지 않는다.
- 설정 변경 전 관련 Compose/FRP/Whale 코드와 문서를 함께 검색한다.
- 일회성 서버 값과 재사용 가능한 프로젝트 설정을 구분한다.
- 검증된 수동 절차만 스크립트로 자동화한다.
- 스크립트는 가능하면 재실행해도 안전한 형태로 작성한다.
- 스크립트에 호스트 IP, token, 비밀번호, 현재 서버 전용 포트를 하드코딩하지 않는다.
- 기존 코드 스타일과 파일 구조를 따른다.
- 변경 후 `git diff --check`와 관련 설정 검증을 실행한다.
- 수행한 작업·변경 대상·검증 결과·남은 항목·복구 방법은 `SOW/`의 날짜별 작업 명세에 기록한다. 비밀값은 포함하지 않는다.

권장 문서 및 스크립트 구조:

- `docs/staging-deploy.md`: 새 서버 배포 절차
- `docs/staging-validation.md`: 실제 검증 결과와 증거
- `.env.example`: 필요한 환경변수 목록
- `conf/frp/frpc.toml.example`: 비밀값 없는 템플릿
- `scripts/bootstrap-host.sh`: 검증된 호스트 선행조건 자동화
- `scripts/deploy.sh`: Compose 배포
- `scripts/verify-staging.sh`: 비밀값을 노출하지 않는 상태 점검

현재 저장소 구조가 다르면 기존 구조를 우선하고 불필요하게 새 디렉터리를 만들지 않는다.

## 10. 테스트 및 완료 원칙

변경이나 테스트를 완료했다고 말하려면 다음을 명시한다.

- 실행한 검증
- 확인한 정상 결과
- 실패하거나 건너뛴 검증
- 사용자 환경에서 추가로 확인해야 할 항목
- 변경된 파일 목록
- 데이터 또는 서비스에 미친 영향

동적 문제 인프라의 최종 완료 기준:

- CTFd, DB, cache, Nginx, FRPS, FRPC가 정상 상태
- CTFd 컨테이너에서 Docker API 접근 성공
- CTFd 컨테이너에서 FRPC 관리 API 접근 성공
- 참가자가 `dynamic_docker` 인스턴스 생성 가능
- Swarm service와 task가 정상 실행
- `$FLAG`가 노출 없이 정상 주입되고 실제 문제 동작으로 검증
- 할당된 외부 포트로 서버 내부 및 외부 접속 가능
- 두 참가자의 인스턴스와 플래그가 서로 격리
- 수동 종료와 timeout 자동 종료 후 고아 service/proxy가 없음
- 재시작 또는 장애 복구 검증 결과가 문서화
- 새 서버에서 필요한 수동 선행조건이 배포 문서에 반영

## 11. 현재 작업 파일 관리

- `CURRENT_TASK.md`의 체크리스트를 진행하면서 완료 여부와 증거를 갱신한다.
- 현재 작업이 끝나면 결과를 `docs/staging-validation.md` 등 영구 문서로 옮긴다.
- 다음 작업을 시작할 때는 `AGENTS.md`를 유지하고 `CURRENT_TASK.md`만 새 목표로 교체한다.

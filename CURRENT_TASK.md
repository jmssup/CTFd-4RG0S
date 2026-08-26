# 현재 작업: Linux 스테이징 동적 컨테이너 실증

## 1. 현재 상태

기준일: 2026-08-18

- Linux 스테이징 서버에서 Docker Compose 스택 생성에 성공했다.
- Docker Swarm을 초기화했다.
- `ctfd_frp-containers` attachable overlay network를 생성했다.
- Swarm node label `name=linux-1`을 지정했다.
- Git에서 제외된 `frpc.toml`을 준비했다.
- 기존 Nginx가 `8080`을 사용하고 있고 `18080`도 사용 중이어서 FRPS HTTP 포트를 다른 빈 포트로 조정했다.
- Direct Port Range는 기존 `10000-10100`에서 `10100-10200`으로 변경했다.
- 구형 Compose v1의 `ContainerConfig` 오류를 피하기 위해 Compose v2를 사용한다.
- 아직 Linux 환경의 엔드투엔드 동적 문제 테스트는 완료하지 않았다.

## 1.1. 환경 경계

이 문서의 실행 대상은 **Linux 스테이징 서버의 저장소와 Docker Engine**이다. 이 문서에 적힌 `10100-10200` 및 Linux에서 선택한 FRPS HTTP 포트를 Mac 로컬 설정에 적용하지 않는다.

현재 알고 있는 차이:

| 항목 | Mac 로컬 | 이번 Linux 스테이징 |
| --- | --- | --- |
| Direct Port Range | `10000-10100`으로 실증한 이력 | `10100-10200` |
| FRPS HTTP 호스트 포트 | `8080`을 사용한 이력. 실제 Mac 파일로 재확인 필요 | `8080`, `18080`은 점유 중이므로 현재 Linux 설정에서 선택한 별도 포트 |
| Docker | Docker Desktop/macOS | Docker Engine/Linux, 단일 노드 Swarm |
| 역할 | 이미 성공한 동작의 비교 기준 | 새 서버 재현성과 실제 배포 후보 검증 |

따라서 이번 테스트는 두 환경의 포트 번호가 같은지를 확인하는 작업이 아니다. 다음 논리 관계가 Linux 안에서 일치하는지를 확인한다.

- Linux Compose port mapping
- Linux `frps.toml`
- Linux CTFd-Whale 관리자 설정
- Linux 방화벽과 외부 접속 경로

Mac과 Linux의 차이가 의도된 것인지 결과 문서에 기록한다. 현재 테스트 도중 공통 Compose 파일을 수정해야 한다면 해당 변경이 Mac을 깨뜨리는지 먼저 평가하고, 환경별 `.env` 또는 Compose override 분리는 이번 실증 후 별도 개선사항으로 제안한다.

## 2. 이번 작업의 목표

깨끗하게 재배포한 Linux 스테이징 환경에서 CTFd-Whale 동적 문제의 전체 생명주기를 검증한다.

검증 대상:

1. 기반 컨테이너와 Swarm/overlay network 상태
2. CTFd에서 Docker API 및 FRPC 관리 API 접근
3. Whale 설정과 실제 Compose/FRP 설정의 일치
4. 참가자의 동적 문제 생성
5. Swarm service와 FRP proxy 생성
6. 외부 TCP 접속 및 동적 FLAG 확인·제출
7. 참가자 간 격리
8. 수동 종료와 timeout 자동 삭제
9. 고아 service/proxy가 남지 않는지 확인
10. 안전한 범위의 재시작 및 복구 검증
11. 재현 가능한 배포·검증 문서 작성

이번 작업에서는 UI/테마 개발, 도메인/HTTPS 적용, 대규모 보안 강화는 진행하지 않는다.

## 3. Codex의 실행 원칙

- 먼저 `AGENTS.md`를 읽고 그 안전수칙을 따른다.
- 처음에는 파일을 수정하지 말고 실제 상태를 읽기 전용으로 조사한다.
- 명령을 실행하기 전에 무엇을 확인하는지 한두 문장으로 설명한다.
- 사용자가 직접 해야 하는 웹 UI 동작이 나오면 정확한 메뉴와 확인값을 제시하고 기다린다.
- 토큰, 비밀번호, FLAG 값은 출력하거나 결과 문서에 기록하지 않는다.
- 테스트 실패 시 바로 설정을 바꾸지 말고 관측 증거로 실패 계층을 먼저 구분한다.
- 전체 스택 중지, Nginx 변경, 방화벽 변경, 서버 재부팅은 사전 승인을 받는다.
- commit 또는 push는 요청받기 전까지 하지 않는다.

## 4. 0단계: 저장소와 작업 상태 확인

다음을 확인한다.

```bash
pwd
git status --short
uname -s
uname -m
hostname
docker context show
docker compose version
docker compose config --services
```

추가로 다음 파일의 존재와 관련 설정을 확인하되 비밀값을 출력하지 않는다.

- `docker-compose.yml` 또는 실제 Compose 파일
- `conf/frp/frps.toml`
- `conf/frp/frpc.toml`
- `.env`
- `CTFd/plugins/ctfd-whale`

해야 할 일:

- 실제 Compose 파일명이 다르면 이후 명령에 반영한다.
- 현재 환경이 Linux 스테이징임을 확인하고 Mac Docker context를 잘못 바라보고 있지 않은지 확인한다.
- 기존 사용자 변경사항이 있으면 목록을 요약하고 보존한다.
- CTFd-Whale의 현재 commit 또는 설치 버전을 확인해 기록한다.
- 테스트에 사용할 이미지와 내부 포트가 저장소 또는 문서에 있는지 찾는다.

중단 조건:

- 저장소 위치가 아니거나 필요한 설정 파일이 없음
- 사용자 변경사항과 이번 작업이 충돌함
- 실제 Compose v2를 사용할 수 없음

## 5. 1단계: 기반 인프라 상태 점검

서비스 이름을 확인한 뒤 아래 검사를 실행한다.

```bash
docker compose ps
docker info --format 'Swarm={{.Swarm.LocalNodeState}}'
docker network inspect ctfd_frp-containers \
  --format 'driver={{.Driver}} scope={{.Scope}} attachable={{.Attachable}}'
docker node inspect self --format 'labels={{json .Spec.Labels}}'
docker compose logs --tail=100 frps frpc
```

정상 기준:

- 필요한 Compose 서비스가 모두 실행 상태
- `Swarm=active`
- `driver=overlay scope=swarm attachable=true`
- node labels에 `"name":"linux-1"`
- FRPC가 FRPS에 정상 로그인
- 반복되는 인증·연결·재시작 오류가 없음

`docker service ls`는 이 시점에 비어 있어도 정상이다. 아직 참가자가 문제 인스턴스를 시작하지 않았기 때문이다.

실패하면 전체 스택을 재시작하지 말고 해당 서비스의 상태, exit code, 최근 로그부터 조사한다.

## 6. 2단계: CTFd 내부 연결 점검

실제 CTFd Compose 서비스 이름을 사용해 Docker API를 확인한다.

예시:

```bash
docker compose exec ctfd python -c \
"import docker; v=docker.from_env().version(); print('Docker API OK:', v.get('ApiVersion'), v.get('Version'))"
```

FRPC 관리 포트 연결을 확인한다.

```bash
docker compose exec ctfd python -c \
"import socket; s=socket.create_connection(('frpc',7400),3); print('FRPC API reachable'); s.close()"
```

정상 기준:

- Docker SDK가 Engine 버전 정보를 반환
- `frpc:7400` TCP 연결 성공
- CTFd/Whale 로그에 Docker API 또는 FRPC API 연결 오류가 없음

Docker API 실패 시 진단 순서:

1. `/var/run/docker.sock` 존재
2. CTFd 컨테이너 mount 여부
3. socket 권한 및 실행 사용자
4. 컨테이너 내부 Unix socket 연결
5. Python Docker SDK 버전과 Docker Engine 최소 API 버전

`DOCKER_API_VERSION`을 먼저 강제하지 않는다.

## 7. 3단계: 포트와 Whale 설정 일치 확인

비밀값을 제외하고 실제 유효 설정을 조사한다.

확인할 값:

- Direct Port Range: `10100-10200`
- FRPS HTTP 가상호스트 포트: 현재 파일에서 발견한 실제 빈 포트
- FRPC API: `http://frpc:7400`
- Auto Connect Network: `ctfd_frp-containers`
- Swarm Node Label: `name=linux-1`

다음 세 위치의 Direct Port Range가 일치해야 한다.

1. Compose의 FRPS host/container port mapping
2. `frps.toml`의 허용 포트 범위
3. Whale 관리자 설정의 Direct Minimum/Maximum Port

다음 세 위치의 HTTP 포트도 일치해야 한다.

1. Compose FRPS HTTP port mapping
2. `frps.toml`의 HTTP 가상호스트 포트
3. Whale 관리자 설정의 외부 HTTP 포트

Whale 관리자 UI는 사용자가 확인해야 할 수 있다. 이 경우 사용자에게 다음 값을 안내한다.

```text
Router Type: frp
FRP API URL: http://frpc:7400
Direct Minimum Port: 10100
Direct Maximum Port: 10200
External HTTP Port: <실제 설정에서 확인한 값>
Auto Connect Network: ctfd_frp-containers
Swarm Node Label: name=linux-1
```

관리자 UI에 빨간 연결 오류가 있으면 다음 단계로 넘어가지 않는다.

## 8. 4단계: 테스트 문제 준비

Mac에서 이미 실증한 것과 **동일한 테스트 이미지, image tag 또는 digest, 내부 포트, 문제 설정**을 사용한다.

주의:

- `ubuntu:22.04` 같은 기본 이미지는 자체적으로 TCP listener를 제공하지 않으므로 `nc` 실증용으로 사용하지 않는다.
- 검증된 이미지 정보가 저장소나 문서에서 발견되지 않으면 추측하지 말고 사용자에게 이미지 이름과 내부 포트를 요청한다.
- 가능하면 tag뿐 아니라 image digest도 기록한다.
- 테스트 timeout은 자동 삭제 검증을 위해 충분히 짧게 설정하되 사용자와 합의한다.

인스턴스 시작 전 기준 상태를 기록한다.

```bash
docker service ls
docker compose logs --tail=50 ctfd frpc frps
```

## 9. 5단계: 단일 참가자 생성·접속·플래그 검증

사용자가 일반 참가자 계정으로 로그인하고 테스트 문제에서 `Start Challenge`를 누르게 한다.

시작 직후 다음을 확인한다.

```bash
docker service ls
docker service ps <생성된_SERVICE> --no-trunc
docker service inspect <생성된_SERVICE> --pretty
```

정상 기준:

- 새로운 Swarm service 생성
- task가 대상 노드에서 `Running`
- 재시작 루프나 image pull 오류 없음
- CTFd UI에 `10100-10200` 범위의 연결 포트 표시
- FRPC/FRPS 로그 또는 상태에서 대응 proxy 확인

FLAG 환경변수는 실제 값을 노출하지 않고 존재 여부만 확인한다.

예시:

```bash
docker service inspect <생성된_SERVICE> \
  --format '{{range .Spec.TaskTemplate.ContainerSpec.Env}}{{println .}}{{end}}' \
  | sed -n 's/^FLAG=.*/FLAG injected/p'
```

접속 검증:

1. Linux 서버 내부에서 할당 포트로 연결
2. 사용자의 Mac 등 서버 외부 네트워크에서 같은 포트로 연결
3. 문제 프로그램이 반환한 FLAG를 CTFd에 제출
4. CTFd가 정답 처리하고 점수·solve 기록을 반영하는지 확인

외부 클라이언트 예시:

```bash
nc <STAGING_SERVER_IP_OR_DOMAIN> <ASSIGNED_PORT>
```

외부 접속 테스트는 실제 외부 네트워크 경로와 방화벽을 검증하므로 서버 내부 테스트로 대체하지 않는다.

## 10. 6단계: 참가자 격리 검증

가능하면 두 개의 일반 참가자 계정을 사용한다.

확인 항목:

- 각 계정에 별도 Swarm service 또는 별도 인스턴스가 생성됨
- 각 계정에 서로 다른 외부 포트가 할당됨
- 한 참가자의 인스턴스 종료가 다른 참가자에게 영향을 주지 않음
- 동적 FLAG가 계정 또는 인스턴스마다 다름
- 다른 참가자의 FLAG로 제출했을 때 정답 처리되지 않음

FLAG 값 자체는 결과 문서에 기록하지 않는다.

## 11. 7단계: 수동 종료와 timeout 자동 삭제

먼저 한 인스턴스를 UI에서 수동 종료한다.

확인 항목:

- 해당 Swarm service 삭제
- 해당 FRP proxy 삭제
- 할당 포트 회수
- 다른 참가자의 인스턴스는 계속 정상

다음으로 짧은 timeout을 설정한 전용 테스트 인스턴스를 생성하고 만료를 기다린다.

확인 항목:

- timeout 이후 service가 자동 삭제됨
- UI 상태가 종료 상태로 갱신됨
- FRP proxy와 포트가 회수됨
- CTFd, Whale, FRPC 로그에 반복 오류 없음

만료 직후 다음을 점검한다.

```bash
docker service ls
docker compose logs --since=10m ctfd frpc frps
```

## 12. 8단계: 재시작·복구 검증

이 단계는 서비스에 영향을 줄 수 있으므로 먼저 사용자 승인을 받는다.

검증 후보:

- CTFd 컨테이너만 재시작
- FRPC/FRPS 재시작
- 전체 Compose 스택의 안전한 재기동
- Docker daemon 또는 서버 재부팅 후 자동 복구

한 번에 하나만 수행하며 각 단계 전후로 활성 인스턴스, Swarm service, proxy, 웹 접근 상태를 비교한다.

서버 재부팅은 마지막에 별도 승인 후 진행한다. `down -v`, prune, Swarm 재초기화는 사용하지 않는다.

## 13. 실패 유형별 진단 방향

### Start Challenge 후 service가 생성되지 않음

- CTFd/Whale 로그
- Docker API 접근
- node label 조건
- image 이름 및 pull 권한
- overlay network 존재 여부

### service는 Running이지만 외부 포트 연결 실패

- UI가 할당한 포트와 현재 Direct Port Range
- Compose host port mapping
- FRPS 허용 포트
- FRPC proxy 생성 여부
- 문제 컨테이너의 실제 listener와 내부 포트
- 호스트 및 외부 방화벽

### `nc` 연결은 되지만 아무 출력도 없음

- 문제 프로그램이 접속 즉시 데이터를 출력하는 구조인지 확인
- 컨테이너 또는 service 로그 확인
- 내부 listener 프로세스와 bind address 확인
- FLAG 환경변수 존재 여부를 값 노출 없이 확인
- 서비스 내부에서 직접 접속해 FRP 계층과 문제 이미지 계층 분리

FRP 문제라고 즉시 단정하지 않는다.

### 포트가 이미 사용 중임

- `ss -ltnp`
- `docker ps`의 publish 정보
- `docker service ls`
- 기존 Nginx와 공유 서버 서비스

점유자를 확인하지 않고 프로세스를 종료하지 않는다.

## 14. 결과 기록

검증 중 또는 종료 후 `docs/staging-validation.md`를 작성하거나 갱신한다.

권장 형식:

```markdown
# Linux staging validation

## Environment
- OS/architecture:
- Docker Engine:
- Docker Compose:
- CTFd commit:
- CTFd-Whale commit:
- FRP image/version:

## Effective configuration
- Overlay network:
- Node label:
- Direct port range:
- HTTP vhost port:

## Test results
| Test | Result | Evidence | Notes |
| --- | --- | --- | --- |
| Compose services | PASS/FAIL | command/log | |
| Docker API | PASS/FAIL | masked output | |
| FRPC API | PASS/FAIL | masked output | |
| Dynamic spawn | PASS/FAIL | service/task | |
| External TCP | PASS/FAIL | client result | |
| Dynamic flag | PASS/FAIL | no secret | |
| Isolation | PASS/FAIL | two instances | |
| Manual cleanup | PASS/FAIL | no orphan | |
| Timeout cleanup | PASS/FAIL | no orphan | |
| Restart recovery | PASS/FAIL/SKIP | | |

## Discovered prerequisites

## Failures and fixes

## Remaining risks
```

비밀값, 실제 FLAG, join token, 관리자 계정 정보는 기록하지 않는다.

과거 진행 보고서의 포트 범위 `10000-10100`은 실증 결과가 통과한 뒤 `10100-10200`으로 갱신한다.

## 15. 완료 조건

이번 작업은 다음이 모두 충족되면 완료다.

- [ ] 기반 Compose/Swarm/overlay 상태 정상
- [ ] CTFd에서 Docker API 접근 성공
- [ ] CTFd에서 FRPC 관리 API 접근 성공
- [ ] Compose, FRP, Whale 포트·network·label 설정 일치
- [ ] 일반 참가자가 동적 인스턴스 생성 가능
- [ ] Swarm task가 안정적으로 Running
- [ ] 서버 외부에서 할당 포트 접속 성공
- [ ] 동적 FLAG 확인 및 CTFd 정답 제출 성공
- [ ] 두 참가자 인스턴스와 FLAG 격리 성공
- [ ] 수동 종료 후 service/proxy/port 회수
- [ ] timeout 자동 종료 후 고아 리소스 없음
- [ ] 승인된 재시작·복구 검증 완료 또는 명시적으로 보류
- [ ] 검증 결과와 새 서버 선행조건 문서화

완료 보고에는 반드시 다음을 포함한다.

1. 통과한 항목
2. 실패 또는 보류한 항목
3. 변경한 파일
4. 실행한 주요 검증
5. 남은 위험과 다음 권장 작업

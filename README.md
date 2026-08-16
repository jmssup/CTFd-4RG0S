# ![](https://github.com/CTFd/CTFd/blob/master/CTFd/themes/core/static/img/logo.png?raw=true)

![CTFd MySQL CI](https://github.com/CTFd/CTFd/workflows/CTFd%20MySQL%20CI/badge.svg?branch=master)
![Linting](https://github.com/CTFd/CTFd/workflows/Linting/badge.svg?branch=master)
[![MajorLeagueCyber Discourse](https://img.shields.io/discourse/status?server=https%3A%2F%2Fcommunity.majorleaguecyber.org%2F)](https://community.majorleaguecyber.org/)
[![Documentation Status](https://api.netlify.com/api/v1/badges/6d10883a-77bb-45c1-a003-22ce1284190e/deploy-status)](https://docs.ctfd.io)

## What is CTFd?

CTFd is a Capture The Flag framework focusing on ease of use and customizability. It comes with everything you need to run a CTF and it's easy to customize with plugins and themes.

![CTFd is a CTF in a can.](https://github.com/CTFd/CTFd/blob/master/CTFd/themes/core/static/img/scoreboard.png?raw=true)

## Features

- Create your own challenges, categories, hints, and flags from the Admin Interface
  - Dynamic Scoring Challenges
  - Unlockable challenge support
  - Challenge plugin architecture to create your own custom challenges
  - Static & Regex based flags
    - Custom flag plugins
  - Unlockable hints
  - File uploads to the server or an Amazon S3-compatible backend
  - Limit challenge attempts & hide challenges
  - Automatic bruteforce protection
- Individual and Team based competitions
  - Have users play on their own or form teams to play together
- Scoreboard with automatic tie resolution
  - Hide Scores from the public
  - Freeze Scores at a specific time
- Scoregraphs comparing the top 10 teams and team progress graphs
- Markdown content management system
- SMTP + Mailgun email support
  - Email confirmation support
  - Forgot password support
- Automatic competition starting and ending
- Team management, hiding, and banning
- Customize everything using the [plugin](https://docs.ctfd.io/docs/plugins/overview) and [theme](https://docs.ctfd.io/docs/themes/overview) interfaces
- Importing and Exporting of CTF data for archival
- And a lot more...

## Install

The easiest way to get started is to use a [Hosted CTFd instance](https://ctfd.io/pricing/).

For self-hosted installations, check out the [CTFd docs](https://docs.ctfd.io/) for [deployment options](https://docs.ctfd.io/docs/deployment/installation) and the [Getting Started](https://docs.ctfd.io/tutorials/getting-started/) guide.

Below is a summary of a few options for self-hosting CTFd:

### Docker

You can use Docker Compose with the following command from the source repository:

`docker compose up`

To use the auto-generated Docker images directly:

`docker run -p 8000:8000 -it ctfd/ctfd`

### CTFd-Whale 연동 메모

이 저장소는 CTFd 오픈소스에 [CTFd-Whale](https://github.com/frankli0324/ctfd-whale) 플러그인을 `CTFd/plugins/ctfd-whale` 경로로 연결한 구성입니다. `docker-compose.yml`은 CTFd, nginx, MariaDB, Redis, frps, frpc를 함께 실행하며 CTFd 컨테이너가 호스트 Docker Engine을 제어할 수 있도록 `/var/run/docker.sock`을 마운트합니다.

#### 실행 전 준비

1. `.env.example`을 기준으로 `.env`를 만들고 `FRP_TOKEN`을 배포 환경에 맞게 변경합니다.
2. Docker Swarm을 초기화하고 Whale이 사용할 노드 라벨을 설정합니다.

```sh
docker swarm init
docker node update --label-add "name=linux-1" $(docker node ls -q)
```

3. challenge 컨테이너가 frpc와 붙을 overlay 네트워크를 미리 생성합니다.

```sh
docker network create --driver overlay --attachable ctfd_frp-containers
```

4. 이미지를 다시 빌드하고 서비스를 올립니다.

```sh
docker compose up -d --build
```

`whale-test/Dockerfile`은 Whale 연동 확인용 challenge 이미지입니다. 단일 노드 테스트에서는 아래처럼 로컬 이미지를 빌드한 뒤 challenge의 Docker Image 값으로 사용할 수 있습니다.

```sh
docker build -t whale-test:latest whale-test
```

멀티 노드 Swarm에서는 각 노드가 같은 이미지를 받을 수 있도록 registry에 push한 이미지명을 challenge에 설정해야 합니다.

#### Whale 관리자 설정 예시

CTFd 초기 설정 후 `/plugins/ctfd-whale/admin/settings`에서 다음 값을 기준으로 설정합니다.

- Docker API URL: `unix:///var/run/docker.sock`
- Docker Swarm Nodes: `linux-1`
- Auto Connect Network: `ctfd_frp-containers`
- FRP API URL: `http://frpc:7400`
- Direct Minimum Port / Direct Maximum Port: `10000` / `10100`
- HTTP Port: `8080`
- Direct IP Address와 HTTP Domain Suffix는 참가자가 접근할 서버 IP 또는 도메인에 맞춥니다.

#### Docker SDK / Docker Engine 호환성

CTFd-Whale 플러그인의 Python 의존성은 `CTFd/plugins/ctfd-whale/requirements.txt`에서 `docker==4.1.0`으로 고정되어 있습니다. Docker SDK와 Docker Engine API 협상 문제를 피하기 위해 `CTFd/plugins/ctfd-whale/utils/docker.py`에서 SSL 사용 여부와 관계없이 `docker.DockerClient(..., version="1.44")`를 명시하도록 수정했습니다.

운영 서버의 Engine API가 이 값보다 낮으면 Whale이 Docker에 연결하지 못할 수 있습니다. 배포 전 호스트에서 아래 명령으로 API 버전을 확인하고, 낮은 경우 Docker Engine을 업그레이드하거나 `utils/docker.py`의 API 버전 핀을 해당 Engine이 지원하는 값으로 맞춥니다.

```sh
docker version --format '{{.Server.APIVersion}}'
```

`docker-compose.yml`에는 CTFd 컨테이너 환경변수로 `DOCKER_API_VERSION=auto`도 추가되어 있습니다. 단, Whale 플러그인 내부 Docker SDK 호출은 위의 `version="1.44"` 설정을 우선 기준으로 사용합니다.

#### FRP 버전 호환성

현재 compose 구성은 `snowdreamtech/frps:latest`, `snowdreamtech/frpc:latest`와 `conf/frp/*.toml` 설정 파일을 사용합니다. 최신 FRP는 기존 INI 스타일 proxy 블록 대신 TOML 형식의 `[[proxies]]`와 `localIP`, `localPort`, `remotePort`, `transport.useCompression` 키를 사용하므로 `CTFd/plugins/ctfd-whale/utils/routers/frp.py`를 이에 맞게 수정했습니다.

FRP 이미지를 구버전으로 바꾸거나 플러그인을 원본으로 되돌리면 이 설정 형식이 다시 맞지 않을 수 있으니, FRP 이미지 버전과 Whale의 proxy config 생성 로직을 함께 확인해야 합니다.

### Development

1. Install [uv](https://docs.astral.sh/uv/getting-started/installation/) and run `uv sync` to create a virtualenv with the dependencies installed.
   - You can also install into an existing environment with `pip install -r requirements.txt`, which is generated from `uv.lock`.
2. Modify [CTFd/config.ini](https://github.com/CTFd/CTFd/blob/master/CTFd/config.ini) to your liking.
3. Use `uv run serve.py` or `uv run flask run` in a terminal to drop into debug mode.

## Live Demo

https://demo.ctfd.io/

## Support

To get basic support, you can join the [MajorLeagueCyber Community](https://community.majorleaguecyber.org/): [![MajorLeagueCyber Discourse](https://img.shields.io/discourse/status?server=https%3A%2F%2Fcommunity.majorleaguecyber.org%2F)](https://community.majorleaguecyber.org/)

If you prefer commercial support or have a special project, feel free to [contact us](https://ctfd.io/contact/).

## Managed Hosting

Looking to use CTFd but don't want to deal with managing infrastructure? Check out [the CTFd website](https://ctfd.io/) for managed CTFd deployments.

## MajorLeagueCyber

CTFd is heavily integrated with [MajorLeagueCyber](https://majorleaguecyber.org/). MajorLeagueCyber (MLC) is a CTF stats tracker that provides event scheduling, team tracking, and single sign on for events.

By registering your CTF event with MajorLeagueCyber users can automatically login, track their individual and team scores, submit writeups, and get notifications of important events.

To integrate with MajorLeagueCyber, simply register an account, create an event, and install the client ID and client secret in the relevant portion in `CTFd/config.py` or in the admin panel:

```python
OAUTH_CLIENT_ID = None
OAUTH_CLIENT_SECRET = None
```

## Credits

- Logo by [Laura Barbera](http://www.laurabb.com/)
- Theme by [Christopher Thompson](https://github.com/breadchris)
- Notification Sound by [Terrence Martin](https://soundcloud.com/tj-martin-composer)

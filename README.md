# WINS CTF — GZCTF 배포 세팅

교육생들이 만든 CVE 기반 CTF 문제를 **동적 컨테이너**로 호스팅하기 위한 [GZCTF](https://github.com/GZTimeWalker/GZCTF) 배포 패키지입니다.
플랫폼 본체(웹/DB) + 문제 컨테이너 오케스트레이션을 `docker compose` 한 벌로 올립니다.

> **flag 주입 방식**: 참가자가 인스턴스를 생성하면 GZCTF가 **인스턴스마다 다른 flag**를 `GZCTF_FLAG` 환경변수로 문제 컨테이너에 주입합니다. 문제 이미지의 entrypoint 가 이 값을 받아 flag 파일로 씁니다. (flag 공유/유출 방지)

---

## ⚠️ 넘겨받으신 분께 (먼저 읽어주세요)

1. **반드시 본인 로컬(PC)에서 한 번 띄워보고**, 아래 「빠른 시작」대로 문제 하나 등록 → 인스턴스 생성 → 풀이 제출까지 **직접 확인한 뒤** 실서버에 올려주세요. 로컬 검증 없이 바로 실서버로 가면 포트/방화벽/이미지 문제로 시간 날립니다.
2. 시크릿(비밀번호·키)은 이 레포에 들어있지 않습니다. `.env.example` → `.env` 로 복사해서 **직접 생성**해야 합니다.
3. GZCTF 서버는 **팀 공용 1대**를 상시 켜두는 구조입니다(클라우드 VM 권장). 각자 띄우는 게 아닙니다.

---

## 구성

```
gzctf/
├── docker-compose.yml    # gzctf(v1.8.7 고정) + postgres. 시크릿은 .env 에서 주입
├── appsettings.json      # 플랫폼 설정 (비밀값 없음 — env 가 덮어씀)
├── .env.example          # 시크릿 템플릿 → .env 로 복사해 채우기
├── .gitignore            # .env, data/ 커밋 차단
└── data/                 # (런타임 생성) DB·업로드 파일. 커밋 안 함
```

---

## 빠른 시작 (로컬 테스트)

전제: Docker Desktop(또는 Docker Engine) + compose 설치.

```bash
git clone <이 레포 주소>
cd gzctf

cp .env.example .env
# .env 를 열어 DB_PASSWORD / GZCTF_ADMIN_PASSWORD / XOR_KEY 를 랜덤 값으로 채운다.
# PUBLIC_ENTRY 는 로컬 테스트면 localhost 그대로.

docker compose up -d
```

- 웹 UI: <http://localhost:8080>
- 관리자 로그인: 아이디 `Admin`, 비번 = `.env` 의 `GZCTF_ADMIN_PASSWORD`
- 종료: `docker compose down` (데이터 유지) / `docker compose down -v` 는 볼륨까지 삭제이니 주의

> **문제 등록 화면은 넓은 화면이 필요합니다.** 창이 좁으면 "화면이 충분히 넓지 않습니다"가 뜨니 브라우저를 최대화하세요.

---

## 실서버 배포 (OCI Always Free VM)

오라클 클라우드 무료 티어 VM은 데이터센터에서 24시간 켜져 있으므로, 한 번 올려두면 상시 운영됩니다.

### 1. VM 생성
1. [OCI 가입](https://www.oracle.com/kr/cloud/free/) (카드 인증만, 과금 없음).
2. Compute → Instances → **Create instance**
   - Image: **Ubuntu 22.04**
   - Shape: **Ampere A1 (Always Free, ARM)** 권장 — 무료로 최대 4 OCPU / 24GB. (선택 불가 시 `VM.Standard.E2.1.Micro` x86)
   - SSH 공개키 등록 (접속용)

### 2. 네트워크 개방 (가장 자주 막히는 부분)
VCN → Security List → **Ingress Rules** 에 추가하고, 서버 안 방화벽(`ufw`/`iptables`)에서도 동일하게 엽니다.

| 포트 | 용도 |
|---|---|
| `22/tcp` | SSH |
| `8080/tcp` | 웹 UI (리버스프록시로 443 쓰면 그쪽) |
| `32768–60999/tcp` | **동적 컨테이너 포트 범위** — GZCTF가 문제 컨테이너를 호스트의 임의 포트(도커 기본 ephemeral 범위)로 매핑함. 이 범위를 안 열면 참가자가 인스턴스에 접속 불가 |

### 3. Docker 설치 & 기동
```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER   # 재로그인 후 적용

git clone <이 레포 주소> && cd gzctf
cp .env.example .env
# .env 편집: 시크릿 랜덤 생성 + PUBLIC_ENTRY=<서버 공인 IP 또는 도메인>
docker compose up -d
```
- `PUBLIC_ENTRY` 를 서버 공인 IP/도메인으로 꼭 바꾸세요. 이 값이 참가자에게 안내되는 인스턴스 접속 주소가 됩니다(기본 localhost 그대로 두면 참가자가 접속 못 함).
- VM 재부팅돼도 `restart: unless-stopped` 라 자동 복구됩니다.

### 4. (권장) HTTPS
[Caddy](https://caddyserver.com/) 또는 nginx 리버스프록시로 8080 앞단에 TLS를 붙이거나 Cloudflare를 얹으세요.

---

## 설정 레퍼런스

### `.env` (시크릿 — 커밋 금지)
| 키 | 설명 |
|---|---|
| `DB_PASSWORD` | Postgres 비밀번호 (아무 랜덤 문자열) |
| `GZCTF_ADMIN_PASSWORD` | 최초 기동 시 생성되는 `Admin` 계정 비번 |
| `XOR_KEY` | flag 암호화 키 (유출 금지) |
| `PUBLIC_ENTRY` | 참가자에게 안내될 접속 호스트. 로컬=`localhost`, 실서버=공인 IP/도메인 |

### `appsettings.json` (플랫폼 설정 — 비밀값 없음)
| 키 | 설명 |
|---|---|
| `ContainerProvider.Type` | `Docker`(단일 서버) 또는 `Kubernetes` |
| `ContainerProvider.DockerConfig.Uri` | 도커 소켓 경로(`unix:///var/run/docker.sock`) |
| `GlobalConfig.Title` | 사이트 제목 |
| `GlobalConfig.DefaultLifetime` | 인스턴스 기본 수명(분) |
| `EmailConfig` | 이메일 가입/인증을 쓸 때만 채움 (선택) |
| `RegistryConfig` | 문제 이미지를 **사설 레지스트리**에서 pull 할 때만 (선택) |
| `DisableRateLimit` | 소규모 운영이면 `true` 로 둬도 무방 |

> 위 표의 `ConnectionStrings:Database` / `XorKey` / `ContainerProvider:PublicEntry` 세 값은 `docker-compose.yml` 이 `.env` 에서 환경변수로 주입하며 `appsettings.json` 값을 덮어씁니다. 그래서 커밋 파일엔 비밀값이 없습니다.

---

## 문제 등록 (동적 컨테이너)

1. **문제 이미지 준비** — 문제 레포를 clone 해서 빌드. 예:
   ```bash
   git clone https://github.com/Minseo9503/cve-2025-30208.git
   cd cve-2025-30208
   docker build -t chal/cve-2025-30208:v1 .
   ```
   같은 도커 호스트에 이미지가 있으면 GZCTF가 그대로 스폰합니다. 다른 호스트에서 돌린다면 레지스트리에 push 하고 `appsettings.json` 의 `RegistryConfig` 를 채우세요.
   - ARM(Ampere) VM에서 돌릴 때 문제 이미지가 amd64 전용이면 해당 호스트에서 다시 빌드하거나 `docker buildx` 로 멀티아치 빌드가 필요합니다. (cve-2025-30208 은 아키텍처 무관)
2. Admin 로그인 → **게임(대회) 생성** → 해당 게임에 **문제 추가** → 유형 **동적 컨테이너**.
3. 컨테이너 이미지 / 노출 포트 / 메모리 제한 / **flag 템플릿**(예: `WINS{[GUID]}`) 설정 후 문제 활성화.
4. 문제별 상세 배포값(이미지명·내부 포트·메모리·분류 등)은 **각 문제 레포 README의 「배포 정보」 블록**을 참고하세요.

---

## 보안 메모
- 문제 컨테이너에는 도커 소켓/특권 권한을 절대 부여하지 않습니다. 소켓 마운트는 **GZCTF 본체(오케스트레이터)에만** 필요합니다.
- 실제 flag 는 코드/이미지에 하드코딩하지 않고 `GZCTF_FLAG` 로 런타임 주입합니다.
- `XOR_KEY` 와 `.env` 전체는 유출 시 flag가 노출될 수 있으니 저장소·채팅에 올리지 마세요.

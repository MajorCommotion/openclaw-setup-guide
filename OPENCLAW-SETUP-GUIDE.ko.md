# OpenClaw 설정 가이드 - 보안 우선 멀티 채널 구성

> **번역 상태:** 한국어 번역 (2026-04-28) | **원본:** OPENCLAW-SETUP-GUIDE.md

**엔터프라이즈급 보안 관행으로 오픈클로 (OpenClaw)를 배포하기 위한 종합 가이드**

**저자**: 프로덕션 배포 경험을 바탕으로 작성  
**날짜**: 2026년 4월  
**대상 독자**: 고급 사용자, 셀프 호스팅 운영자, 개인정보 보호에 민감한 개인  
**배포 유형**: 엄브렐 (Umbrel) 홈 서버 (Debian 기반)

---

## 목차

1. [철학 및 접근 방식](#philosophy--approach)
2. [설치 전 계획](#pre-installation-planning)
3. [설치](#installation)
4. [초기 구성](#initial-configuration)
5. [보안 강화](#security-hardening)
6. [멀티 채널 설정](#multi-channel-setup)
7. [디스코드 (Discord) 보안 잠금](#discord-security-lockdown)
8. [대체 메시징 플랫폼](#alternative-messaging-platforms)
9. [운영 보안](#operational-security)
10. [비용 최적화](#cost-optimization)
11. [유지 관리 및 모니터링](#maintenance--monitoring)
12. [교훈](#lessons-learned)

---

## 철학 및 접근 방식

### 핵심 원칙

**1. 보안 우선, 편의성은 그 다음**
- 모든 외부 통신은 적대적이라고 가정
- 심층 방어 구현
- 절대 신뢰하지 말고, 항상 검증하라

**2. 설계 단계부터 개인정보 보호**
- 데이터 노출 최소화
- 민감한 작업 격리
- 가능한 경우 로컬 우선 아키텍처 적용

**3. 비용 의식**
- API 사용 최적화 (일상적인 작업에는 무료/저렴한 모델 활용)
- 지출 모니터링 (CRON 작업의 모델 선택이 중요)
- 적절한 경우 로컬 추론 사용 (Ollama)

**4. 문서화 및 재현성**
- 모든 구성 결정 사항을 문서화
- 메모리 시스템이 변화를 추적
- 감사 및 롤백이 용이

---

## 설치 전 계획

### 1. 위협 모델 정의

**스스로에게 물어보세요:**
- 무엇을 보호하려 하는가? (인프라 세부 정보, 개인 데이터, 금융 시스템)
- 누구로부터 보호하려 하는가? (공개 인터넷, 비인가 사용자, 소셜 엔지니어링)
- 어느 정도의 위험이 허용 가능한가? (위험 허용 한도 정의)

**이 배포 환경의 위협 모델:**
- **높은 위험**: 민감한 금융 인프라 노출
- **중간 위험**: 그룹 채팅을 통한 개인 데이터 유출
- **낮은 위험**: API 키 노출 (속도 제한됨, 교체 가능)

### 2. 아키텍처 결정

**핵심 질문:**
1. **단일 서버 또는 분산 구성?**
   - 이 설정: 분산형 (금융 서비스를 AI와 격리)
   - 이유: 보안 > 편의성

2. **공개 또는 비공개?**
   - 이 설정: 비공개 (공개 엔드포인트 없음)
   - 이유: 공격 표면 최소화

3. **어떤 메시징 플랫폼을 사용할 것인가?**
   - 이 설정: 디스코드 (Discord) (기본), 매트릭스 (Matrix) (계획됨), 기타 가능
   - 이유: 접근성과 보안 사이의 균형

### 3. 리소스 계획

**하드웨어 요구 사항:**
- **최소**: 8 GB RAM, CPU 코어 4개, 저장 공간 50 GB
- **권장**: 16 GB RAM, CPU 코어 8개, 저장 공간 100 GB
- **이 배포 환경**: 16 GB RAM (Umbrel Pro)

**API 예산:**
- **예상**: 월 $10~20 (사용량에 따라 다름)
- **최적화 시**: 월 $7~12 (CRON 작업에 Haiku 사용 시)
- **이 배포 환경**: 실제 월 ~$10~15

---

## 설치

### 방법 1: NPM 전역 설치 (권장)

**플랫폼**: Debian/Ubuntu 기반 시스템

```bash
# Install Node.js (v22+ required)
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt-get install -y nodejs

# Install OpenClaw globally
npm install -g openclaw

# Verify installation
openclaw --version
```

### 방법 2: 엄브렐 (Umbrel) 앱 스토어

**엄브렐 (Umbrel) 사용자의 경우:**
1. 엄브렐 (Umbrel) 대시보드 열기
2. 앱 스토어로 이동
3. "OpenClaw" 검색
4. 설치 클릭
5. 컨테이너 시작 대기

**참고**: 엄브렐 (Umbrel) 설치는 미리 정의된 경로가 있는 도커 (Docker) 컨테이너에서 실행됩니다.

---

## 초기 구성

### 1. 최초 실행

```bash
# Start OpenClaw Gateway
openclaw gateway start

# Check status
openclaw status
```

**예상 출력:**
- `http://localhost:18789`에서 Gateway 실행 중
- Control UI 접근 가능
- 아직 구성된 채널 없음

### 2. API 제공자 설정

**권장 순서:**

**1단계: Anthropic (기본)**
```json
{
  "models": {
    "providers": {
      "anthropic": {
        "apiKey": "***",
        "models": [
          {
            "id": "claude-sonnet-4-5",
            "cost": {"input": 3, "output": 15}
          },
          {
            "id": "claude-haiku-4-5",
            "cost": {"input": 1, "output": 5}
          }
        ]
      }
    }
  }
}
```

**비용 최적화 팁**: CRON 작업에는 Haiku를, 주요 상호작용에는 Sonnet을 사용하세요.

**2단계: OpenAI (선택 사항 - 오디오/Whisper)**
```json
{
  "models": {
    "providers": {
      "openai": {
        "apiKey": "***",
        "models": [
          {
            "id": "gpt-5.2",
            "cost": {"input": 1.75, "output": 14}
          }
        ]
      }
    }
  },
  "tools": {
    "media": {
      "audio": {
        "enabled": true,
        "models": [{"provider": "openai", "model": "whisper-1"}]
      }
    }
  }
}
```

**3단계: Ollama (로컬/무료 - 선택 사항)**
```bash
# Install Ollama on your server
curl -fsSL https://ollama.com/install.sh | sh

# Pull models
ollama pull qwen2.5:14b
ollama pull llama3.2:3b
```

```json
{
  "models": {
    "providers": {
      "ollama": {
        "baseUrl": "http://localhost:11434/v1",
        "apiKey": "***",
        "models": [
          {
            "id": "qwen2.5:14b",
            "cost": {"input": 0, "output": 0}
          }
        ]
      }
    }
  }
}
```

### 3. 에이전트 구성

**기본 모델 선택:**
```json
{
  "agents": {
    "defaults": {
      "model": {
        "primary": "anthropic/claude-sonnet-4-5"
      }
    }
  }
}
```

**이유**: 주요 상호작용의 품질을 위해 Sonnet을, 자동화의 비용 절감을 위해 Haiku를 사용합니다.

---

## 보안 강화

### 1. Gateway 구성

**공격 표면 최소화:**

```json
{
  "gateway": {
    "mode": "local",
    "controlUi": {
      "allowInsecureAuth": true,
      "dangerouslyDisableDeviceAuth": true,
      "allowedOrigins": [
        "http://localhost:18789",
        "http://127.0.0.1:18789"
      ]
    },
    "auth": {
      "mode": "token",
      "token": "***"
    }
  }
}
```

**보안 참고 사항:**
- `allowInsecureAuth: true` - 로컬 전용 배포에서만 사용
- `allowedOrigins` - localhost로 제한 (외부 접근 불가)
- 강력한 토큰 생성: `openssl rand -hex 32`

### 2. 상승된 명령 제어

**원칙**: 파괴적인 작업은 신뢰할 수 있는 인터페이스에서만 허용.

```json
{
  "tools": {
    "elevated": {
      "enabled": true,
      "allowFrom": {
        "webchat": ["openclaw-control-ui"]
      }
    }
  }
}
```

**이 설정의 효과:**
- ✅ Control UI (webchat)에서 상승된 명령 실행 가능
- ❌ 디스코드 (Discord) 채널은 상승된 명령 실행 불가
- ❌ 기타 메시징 인터페이스는 기본적으로 차단됨

**사이버 보안 근거:**
- **Control UI** = 신뢰할 수 있는 로컬 인터페이스 (물리적으로 직접 접근)
- **디스코드 (Discord)** = 외부, 잠재적으로 침해 가능 (소셜 엔지니어링 위험)
- **최소 권한 원칙**: 절대적으로 필요한 경우에만 상승된 접근 권한 부여

### 3. 파일 시스템 접근 제어

**작업 공간 경계 생성:**

```bash
# OpenClaw workspace (default)
~/.openclaw/workspace/

# Sensitive directories - RESTRICT ACCESS
# Do NOT grant OpenClaw direct access to:
# - SSH keys (~/.ssh/)
# - financial service data stores (~/.finance/secure-data.dat)
# - payment processing service data (~/.payment-service/)
# - Personal documents (~/)
```

**모범 사례**: 오픈클로 (OpenClaw)는 자체 샌드박스(`~/.openclaw/workspace/`)에서 운영되어야 합니다. 파일 시스템 전체에 대한 무제한 접근 권한을 절대 부여하지 마세요.

---

## 멀티 채널 설정

### 개요

**이 배포 환경에서 사용하는 구성:**
- **Control UI (webchat)**: 소유자 인터페이스, 전체 접근 권한, 신뢰됨
- **디스코드 (Discord)**: 다중 채널, 제한된 권한, 관리됨
- **매트릭스 (Matrix)**: 계획됨 (Umbrel Pro에서 Synapse + Element 운영 중)

### 채널 보안 계층 구조

| 인터페이스 | 신뢰 수준 | 상승 권한 | 인프라 접근 | 사용 사례 |
|---------|-------------|----------|----------------------|----------|
| Control UI | 높음 | ✅ 예 | ✅ 예 | 소유자 관리 |
| Discord #personal | 중간 | ❌ 아니오 | ❌ 아니오 | 소유자 채팅 |
| Discord #general | 낮음 | ❌ 아니오 | ❌ 아니오 | 그룹 채팅 |
| Discord #work | 중간 | ❌ 아니오 | ❌ 아니오 | 업무 주제 |
| Matrix | 미정 | ❌ 아니오 | ❌ 아니오 | 계획됨 |

**핵심 원칙**: 로컬 → 비공개 → 그룹 컨텍스트로 이동할수록 신뢰도는 낮아집니다.

---

## 디스코드 (Discord) 보안 잠금

### 1. 서버 설정 (비공개 서버)

**비공개 디스코드 (Discord) 서버 생성:**
1. 새 디스코드 (Discord) 서버 생성 (소유자로서)
2. 서버를 **비공개** (초대 전용)로 설정
3. 멤버 초대 비활성화 (또는 승인 전용)
4. 관리자 작업에 2FA 요구 사항 활성화

**서버 설정:**
- 개인정보 보호: 비공개 (검색 불가)
- 인증: 최고 수준 (10분 대기 + 이메일 인증 + 전화번호 인증)
- 기본 권한: 최소 (읽기 전용)

### 2. 봇 등록

**디스코드 (Discord) 애플리케이션 생성:**
1. https://discord.com/developers/applications 로 이동
2. New Application 생성 → 이름 지정 (예: "OpenClaw Assistant")
3. **Bot** 섹션으로 이동 → 봇 생성
4. **Privileged Gateway Intents**:
   - ✅ Server Members Intent (사용자 감지에 필수)
   - ✅ Message Content Intent (메시지 읽기에 필수)
   - ❌ Presence Intent (불필요)

**봇 권한 (권장 최소 구성):**
- ✅ Read Messages/View Channels
- ✅ Send Messages
- ✅ Send Messages in Threads
- ✅ Embed Links
- ✅ Attach Files
- ✅ Read Message History
- ✅ Add Reactions
- ❌ Administrator (절대 부여 금지)
- ❌ Manage Server
- ❌ Manage Channels
- ❌ Kick/Ban Members

**초대 링크 생성:**
```
https://discord.com/oauth2/authorize?client_id=YOUR_CLIENT_ID&permissions=412317273088&scope=bot
```

**권한 값**: `412317273088` (읽기, 전송, 임베드, 첨부, 기록, 반응)

### 3. 오픈클로 (OpenClaw) 디스코드 (Discord) 구성

**최소 보안 구성:**

```json
{
  "channels": {
    "discord": {
      "enabled": true,
      "token": "***",
      "groupPolicy": "allowlist",
      "dmPolicy": "pairing",
      "allowFrom": [
        "YOUR_USER_ID_HERE"
      ],
      "guilds": {
        "YOUR_GUILD_ID": {
          "requireMention": false,
          "users": [
            "YOUR_USER_ID_HERE",
            "AUTHORIZED_USER_2_ID"
          ]
        }
      }
    }
  }
}
```

**보안 기능 설명:**

**`groupPolicy: "allowlist"`**
- 봇은 명시적으로 허용된 서버에서만 응답
- 다른 모든 초대/길드 무시
- 비인가 사용 방지

**`dmPolicy: "pairing"`**
- 다이렉트 메시지는 페어링/인가 필요
- DM을 통한 스팸/남용 방지
- 소유자가 사용자를 명시적으로 인가해야 함

**`allowFrom: [...]`**
- 전역 사용자 허용 목록
- 봇은 이 사용자 ID들에게만 응답
- 다른 모든 사용자 조용히 거부

**`guilds.{id}.users: [...]`**
- 서버별 사용자 허용 목록
- 추가적인 접근 제어 레이어
- 전역 허용 목록보다 더 유연하게 설정 가능

**`requireMention: false`**
- @멘션 없이도 봇이 응답
- 자연스러운 대화 가능
- 허용 목록이 이미 접근을 제한하므로 안전

### 4. 채널 수준 격리

**주제별 채널 생성:**

```
YOUR_DISCORD_SERVER/
├── #general (general conversation)
├── #personal (private owner channel)
├── #language-study (language learning)
├── #linux (Linux topics)
├── #aspnet-c-study (C#/ASP.NET study)
└── #work-mes (work-related)
```

**각 채널 = 격리된 오픈클로 (OpenClaw) 세션:**
- 별도의 대화 기록
- 독립적인 컨텍스트
- 교차 오염 없음

**세션 키 형식:**
```
agent:main:discord:channel:<channelId>
```

**채널 ID 가져오기:**
1. 개발자 모드 활성화: 사용자 설정 → 고급 → 개발자 모드 켜기
2. 채널 우클릭 → 채널 ID 복사
3. 구성의 허용 목록에 추가

### 5. 콘텐츠 필터링 및 관리

**디스코드 (Discord) 관리 정책 구현:**

```markdown
# Discord Channel Moderation Rules

**#aspnet-c-study**: C#/ASP.NET/MSSQL/.NET topics only
**#language-study**: Language learning topics only
**#linux**: Linux-related topics only
**#general**: General conversation, redirect specialized topics
**#personal**: No restrictions (owner-only)
**#work-mes**: WinForms + MSSQL focus, all tech questions welcome
```

**봇 동작:**
- 전문 채널에서 주제 벗어난 질문 → 정중하게 리다이렉트
- #general에서 전문 주제 → 적절한 채널로 리다이렉트
- 이중 언어 리다이렉트 사용 (영어 + 사용자 언어)

**리다이렉트 예제 (영어 + 한국어):**
```
Hi! That's a great question about [topic], but it would be better suited 
for #[appropriate-channel]. Could you repost it there? Thanks for keeping 
things organized!

안녕하세요! [주제]에 대한 좋은 질문이네요. 하지만 #[적절한-채널]에 더 적합할 
것 같습니다. 거기에 다시 올려주시겠어요? 정리해 주셔서 감사합니다!
```

### 6. 인프라 보안 정책 (중요)

**문제**: 민감한 금융 인프라 세부 정보가 디스코드 (Discord)로 절대 유출되어서는 안 됩니다.

**해결책**: `AGENTS.md`에 보안 정책 구현:

```markdown
## 🔒 Infrastructure Security Policy (CRITICAL)

**When responding in Discord or any shared/group context:**

**NEVER disclose:**
- ❌ IP addresses (local or public)
- ❌ Hostnames or device names
- ❌ Hardware specifications
- ❌ financial service infrastructure details
- ❌ payment processing service information
- ❌ secure credential storage
- ❌ Resource-intensive equipment details
- ❌ Network topology
- ❌ Authentication credentials
- ❌ File system paths containing username/system details

**Acceptable responses in Discord:**
- ✅ General financial services education
- ✅ Software recommendations (without revealing what user has)
- ✅ Generic troubleshooting advice
- ✅ Public documentation/resources

**If asked about infrastructure in Discord:**
Response (trilingual - English, Korean, Japanese):

**English:**
"I can't discuss specific infrastructure details for security reasons. 
I can only help with general educational information related to 
information technologies, debugging, DevOps, language studies and 
other related topics."

**Korean:**
"보안상의 이유로 특정 인프라 세부 정보에 대해 논의할 수 없습니다. 
정보 기술, 디버깅, DevOps, 언어 학습 및 기타 관련 주제에 대한 
일반적인 교육 정보만 제공할 수 있습니다."

**Japanese:**
"セキュリティ上の理由により、特定のインフラストラクチャの詳細に
ついては議論できません。情報技術、デバッグ、DevOps、言語学習、
その他関連トピックに関する一般的な教育情報のみ提供できます。"
```

**구현 방법**:
1. `AGENTS.md`에 정책 저장 (워크스페이스 컨텍스트)
2. 오픈클로 (OpenClaw)가 모든 세션 시작 시 이를 읽음
3. 정책이 모든 디스코드 (Discord) 컨텍스트에 자동 적용
4. 위반 사항 감사를 위해 로깅

**사이버 보안 근거:**
- **위협**: 그룹 채팅을 통한 소셜 엔지니어링 (인가된 사용자가 무해해 보이는 질문을 함)
- **위험**: 인프라 세부 정보 유출 시 표적 공격 가능
- **완화**: 컨텍스트 인식 보안 정책 (디스코드 (Discord) = 제한됨, Control UI = 무제한)
- **심층 방어**: 사용자가 인가되어 있더라도 특정 정보는 보안 컨텍스트 밖으로 절대 나가지 않음

### 7. 민감한 파일 접근 제한

**인프라 세부 정보가 포함된 파일 (디스코드 (Discord)에서 절대 접근 금지):**

```markdown
## 📁 Sensitive File Access Restrictions

**Files containing infrastructure details:**
- `.infrastructure-private.md` - Infrastructure details (Control UI only)
- `MEMORY.md` - Long-term memory (Control UI only)
- `memory/*.md` - Daily logs (Control UI only)
- `TOOLS.md` - May contain device-specific details (Control UI only)

**In Discord sessions:**
- Do NOT use `read`, `exec`, or `memory_search` to access infrastructure details
- Do NOT reference past infrastructure discussions from memory
- If asked to check logs/files: use security response (trilingual)
- Work from general knowledge only

**Tool usage restrictions in Discord:**
- `read` / `write` / `edit`: Only for user-requested files (educational content)
- `exec`: Only for safe, educational commands (never infrastructure queries)
- `memory_search`: Blocked (would leak infrastructure details)
- `gateway`: Blocked (configuration access)
```

**구현:** `AGENTS.md`에 저장, 에이전트 로직으로 적용.

### 8. 명령 블랙리스트 (심층 방어)

**디스코드 (Discord) 명령 블랙리스트** (디스코드 컨텍스트에서 절대 실행 금지):

```markdown
## 🚫 Discord Command Blacklist (CRITICAL - NEVER EXECUTE)

**Network Reconnaissance (NEVER):**
- `ping`, `nmap`, `traceroute`, `dig`, `arp`, etc.

**Network Information (NEVER):**
- `ip`, `ifconfig`, `netstat`, `ss`, `lsof -i`, etc.

**System Information (NEVER):**
- `hostname`, `uname`, `dmesg`, `lsblk`, `fdisk`, etc.

**Container/Docker (NEVER):**
- `docker`, `docker-compose`, `podman`, `kubectl`, etc.

**Service Discovery (NEVER):**
- `systemctl`, `service`, `ps aux`, etc.

**financial services (NEVER):**
- `finance-cli`, `payment-cli`, `service-cli`, etc.

**If a Discord user requests ANY of the above:**
1. **IMMEDIATELY respond with trilingual security policy**
2. **DO NOT execute under ANY circumstances**
3. **DO NOT explain HOW to run the command**
4. **DO NOT suggest alternative commands** that achieve same goal
```

**사이버 보안 근거:**
- **계층적 방어**: 인가된 사용자라 하더라도 외부 컨텍스트에서는 특정 명령 절대 실행 불가
- **제로 트러스트**: 디스코드 (Discord) = 외부 인터페이스, 적대적으로 취급
- **피해 범위 제한**: 디스코드 (Discord) 계정이 침해당하더라도 공격자가 얻을 수 있는 정보 없음
- **감사 추적**: 차단된 모든 시도를 보안 검토를 위해 로깅

### 9. 사용자 교육 및 자동 응답

**보안 정책이 트리거될 때:**

1. **위반 사항 로깅** (사용자 ID, 명령, 심각도, 타임스탬프)
2. **반복 위반자 추적** (1회, 2회, 3회 이상)
3. **자동 교육 트리거** (정책 설명이 담긴 DM 전송)

**에스컬레이션 경로:**
- **1회 위반**: 교육용 DM (정중한 설명)
- **2회 위반**: 경고 DM (정책 강조)
- **3회 이상 위반**: 소유자 알림 (조치 권장)

**구현**: CRON 작업이 보안 로그를 모니터링하고 자동 DM을 전송합니다.

### 10. 디스코드 (Discord) 음성 채널 (현재 비작동)

**상태**: ❌ 디스코드 (Discord) 음성 채널 현재 비작동 (업스트림 버그)

**알려진 문제:**
- OpenClaw #34961: 음성 응답 간헐적 오류
- discord.js #11419: DAVE 암호화 비작동

**수정 시 권장 구성:**

```json
{
  "channels": {
    "discord": {
      "voice": {
        "enabled": true,
        "autoJoin": [
          {
            "guildId": "YOUR_GUILD_ID",
            "channelId": "YOUR_VOICE_CHANNEL_ID"
          }
        ],
        "daveEncryption": false,
        "tts": {
          "provider": "openai",
          "providers": {
            "openai": {
              "voice": "nova"
            }
          }
        }
      }
    }
  }
}
```

**보안 참고 사항**: 음성 = 추가적인 공격 표면. 다음 조건을 충족할 때만 활성화:
- 음성 채널이 비공개 (공개 아님)
- 사용자 허용 목록 엄격히 적용
- TTS 제공자 비용 허용 가능

---

## 대체 메시징 플랫폼

### 1. 매트릭스 (Matrix) (Synapse + Element)

**상태**: 계획됨 (Umbrel Pro에서 서비스 정상 운영 중, 오픈클로 (OpenClaw) 구성 필요)

**매트릭스 (Matrix)를 선택하는 이유:**
- ✅ 오픈소스, 연합형
- ✅ 기본 종단간 암호화
- ✅ 셀프 호스팅 (완전한 제어)
- ✅ 기업의 데이터 수집 없음
- ❌ 설정이 더 복잡함

**설정 단계:**

**1단계: Synapse 서버 (매트릭스 (Matrix) 홈서버)**
```bash
# Install Synapse via Umbrel App Store
# Or manual installation:
docker run -d \
  --name synapse \
  -v /data/synapse:/data \
  -p 8008:8008 \
  matrixdotorg/synapse:latest
```

**2단계: Element 클라이언트**
```bash
# Install Element via Umbrel App Store
# Or use Element web: https://app.element.io
```

**3단계: 오픈클로 (OpenClaw) 구성**
```json
{
  "channels": {
    "matrix": {
      "enabled": true,
      "homeserverUrl": "https://your-homeserver.tld",
      "accessToken": "YOUR_MATRIX_ACCESS_TOKEN",
      "userId": "@openclaw:your-homeserver.tld",
      "allowFrom": [
        "@owner:your-homeserver.tld"
      ]
    }
  }
}
```

**디스코드 (Discord) 대비 보안 장점:**
- 기본 E2E 암호화
- 셀프 호스팅 (데이터 직접 제어)
- 기업 감시 없음
- 연합형 (탈중앙화)

**보안 고려 사항:**
- **허용 목록 구현 필수** (디스코드 (Discord)와 동일)
- **인프라 접근 제한 필수** (동일한 정책 적용)
- **사용자 인증 필수** (E2E 키 인증)

### 2. 시그널 (Signal)

**시그널 (Signal)을 선택하는 이유:**
- ✅ 업계 최고 수준의 암호화 (Signal 프로토콜)
- ✅ 최소한의 메타데이터 수집
- ✅ 오픈소스
- ❌ 전화번호 필수
- ❌ 제한된 그룹 기능

**Signal-CLI를 통한 설정:**

```bash
# Install signal-cli
wget https://github.com/AsamK/signal-cli/releases/download/v0.12.0/signal-cli-0.12.0.tar.gz
tar xf signal-cli-0.12.0.tar.gz -C /opt
export PATH=$PATH:/opt/signal-cli/bin

# Register number (one-time)
signal-cli -u +YOUR_NUMBER register

# Verify
signal-cli -u +YOUR_NUMBER verify CODE

# Run as daemon
signal-cli -u +YOUR_NUMBER daemon
```

**오픈클로 (OpenClaw) 연동:**
```json
{
  "channels": {
    "signal": {
      "enabled": true,
      "phoneNumber": "+YOUR_NUMBER",
      "allowFrom": [
        "+OWNER_NUMBER"
      ]
    }
  }
}
```

**보안 참고 사항**: 시그널 (Signal) = 가장 개인정보 보호가 강한 옵션이지만, 디스코드 (Discord)/매트릭스 (Matrix)에 비해 기능이 제한적입니다.

### 3. 텔레그램 (Telegram)

**텔레그램 (Telegram)을 선택하는 이유:**
- ✅ 풍부한 기능 (봇, 채널, 그룹)
- ✅ 빠르고 안정적
- ✅ 크로스 플랫폼
- ⚠️ 기본적으로 종단간 암호화 미적용 (비밀 채팅만 해당)
- ⚠️ 클라우드 기반 (텔레그램 (Telegram)이 메시지를 저장)

**BotFather를 통한 설정:**

1. 텔레그램 (Telegram)에서 @BotFather에게 메시지 전송
2. `/newbot` → 봇 이름 지정
3. 봇 토큰 저장
4. 개인정보 보호 모드 설정: `/setprivacy` → Disabled (그룹 메시지 읽기 허용)

**오픈클로 (OpenClaw) 연동:**
```json
{
  "channels": {
    "telegram": {
      "enabled": true,
      "botToken": "YOUR_BOT_TOKEN",
      "allowFrom": [
        YOUR_USER_ID
      ]
    }
  }
}
```

**보안 고려사항:**
- **민감한 인프라에는 적합하지 않음** (클라우드 저장)
- **일반적인 상호작용에만 사용** (금융 서비스 논의 제외)
- **디스코드 (Discord)와 동일한 허용 목록/정책 적용** (접근 제어 제한)

### 4. Slack (Enterprise)

**Slack을 선택하는 이유:**
- ✅ 엔터프라이즈 기능 (SSO, 감사 로그)
- ✅ 광범위한 연동
- ✅ 팀에게 친숙
- ❌ 비용 부담 ($7-15/사용자/월)
- ❌ 클라우드 기반 (Slack이 모든 것을 저장)

**설정:**
1. Slack 앱 생성: https://api.slack.com/apps
2. OAuth 범위 활성화: `chat:write`, `channels:read`, `users:read`
3. 워크스페이스에 설치
4. 봇 토큰 저장

**오픈클로 (OpenClaw) 연동:**
```json
{
  "channels": {
    "slack": {
      "enabled": true,
      "botToken": "***",
      "allowFrom": [
        "OWNER_USER_ID"
      ]
    }
  }
}
```

**보안 참고**: Slack = 엔터프라이즈 컴플라이언스를 제공하지만 시그널 (Signal)/매트릭스 (Matrix)보다 개인정보 보호 수준이 낮습니다.

### 플랫폼 비교

| 플랫폼 | 개인정보 보호 | 보안 | 기능 | 비용 | 최적 용도 |
|--------|-------------|------|------|------|----------|
| **시그널 (Signal)** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐ | 무료 | 최대 개인정보 보호 |
| **매트릭스 (Matrix)** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | 무료 (셀프 호스팅) | 셀프 호스팅 E2E |
| **디스코드 (Discord)** | ⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | 무료 | 풍부한 기능 |
| **텔레그램 (Telegram)** | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | 무료 | 균형 잡힌 선택 |
| **Slack** | ⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | $$$$ | 엔터프라이즈 |

**권장 사항**: 최대 보안을 위해 매트릭스 (Matrix) 또는 시그널 (Signal)을 사용하고, 편의성/기능을 위해 디스코드 (Discord)를 사용하세요.

---

## 운영 보안

### 1. 세션 컨텍스트 인식

**자동 컨텍스트 감지 구현:**

```markdown
## Session Context Detection (CRITICAL)

**Before doing anything:**

Check inbound metadata:
- `channel == "webchat"` → **MAIN SESSION** (full access)
- `channel == "discord"` or other → **RESTRICTED SESSION** (security mode)

**If in MAIN SESSION (Control UI/webchat):**
- ✅ Read MEMORY.md and memory/*.md
- ✅ Execute infrastructure commands
- ✅ Access sensitive files
- ✅ Full tool usage

**If in Discord or group context:**
- ❌ Skip ALL memory files (MEMORY.md, memory/*.md)
- ❌ Never read .infrastructure-private.md
- ❌ Activate infrastructure security policy (trilingual responses)
- ❌ Restrict tool usage (no memory_search, gateway, infrastructure exec)
```

**구현**: `AGENTS.md`에 저장하고, 모든 세션 시작 시 자동으로 적용됩니다.

### 2. 다단계 인증

**Control UI 접근을 위한 설정:**

```bash
# Generate strong token
openssl rand -hex 32

# Store in config
{
  "gateway": {
    "auth": {
      "mode": "token",
      "token": "***"
    }
  }
}
```

**디스코드 (Discord) 봇을 위한 설정:**
- 봇 토큰 (디스코드 (Discord) Developer Portal에서 발급)
- 서버 초대 링크 (권한 제한)
- 사용자 허용 목록 (오픈클로 (OpenClaw) 구성에서 설정)

**계층형 인증:**
1. **디스코드 (Discord) 계정** (비밀번호 + 2FA)
2. **서버 접근** (초대 전용)
3. **봇 허용 목록** (승인된 사용자 ID만)
4. **채널 권한** (디스코드 (Discord) 역할 기반)
5. **오픈클로 (OpenClaw) 정책** (컨텍스트 인식 제한)

### 3. 감사 로깅

**보안 이벤트 모니터링:**

```bash
# Create security log
touch ~/.openclaw/workspace/security-alerts.log
touch ~/.openclaw/workspace/security-audit.log

# Monitor script (runs via CRON)
#!/bin/bash
# Check for infrastructure disclosure attempts
grep -i "finance\|payment-processing\|resource-intensive operations\|IP address" ~/.openclaw/workspace/memory/*.md | \
  grep "discord" >> security-alerts.log
```

**CRON 작업 설정:**
```bash
# Add to CRON (daily at 02:00 UTC)
{
  "name": "Daily Security Audit",
  "schedule": {"kind": "cron", "expr": "0 2 * * *", "tz": "UTC"},
  "payload": {"kind": "systemEvent", "text": "Run daily security audit: /data/.openclaw/workspace/scripts/monitor-discord-security.sh"}
}
```

**기록할 항목:**
- 디스코드 (Discord)에서의 인프라 키워드 언급
- 차단된 명령 시도
- 사용자 허용 목록 위반
- 무단 접근 시도

### 4. 인시던트 대응 계획

**보안 침해가 의심되는 경우:**

1. **즉각 조치**:
   ```bash
   # Disable all external channels
   openclaw gateway stop
   
   # Revoke Discord bot token
   # (Go to Discord Developer Portal → Regenerate Token)
   
   # Check logs
   cat ~/.openclaw/workspace/security-alerts.log
   ```

2. **조사**:
   - `security-alerts.log` 검토 (무엇이 유출되었는가?)
   - `security-audit.log` 검토 (언제 발생했는가?)
   - 침해된 사용자 파악 (해당하는 경우)

3. **복구**:
   - 모든 API 키 교체 (Anthropic, OpenAI 등)
   - 디스코드 (Discord) 봇 토큰 재발급
   - 허용 목록 업데이트 (침해된 사용자 제거)
   - 보안 정책 검토 및 업데이트

4. **사후 처리**:
   - 발생 내용 문서화
   - 보안 정책 업데이트
   - 모니터링에 새로운 탐지 항목 추가

---

#### 비용 최적화

### 1. 모델 선택 전략

**원칙**: 작업을 수행할 수 있는 가장 저렴한 모델을 사용하세요.

| 사용 사례 | 권장 모델 | 비용 | 이유 |
|----------|----------|------|------|
| 주요 상호작용 | Claude Sonnet 4.5 | $3/$15 per MTok | 품질이 중요 |
| CRON 작업 | Claude Haiku 4.5 | $1/$5 per MTok | 80% 저렴, 충분한 성능 |
| 로컬 테스트 | Ollama (Qwen 2.5 14B) | $0 | 무료, 적절한 품질 |
| 오디오 전사 | OpenAI Whisper | $0.006/min | 업계 표준 |

**실제 절감 효과:**
- 최적화 전: $15-20/월 (모든 작업에 Sonnet 사용)
- 최적화 후: $7-12/월 (CRON에 Haiku, 주요 작업에 Sonnet 사용)
- **절감액**: 약 40-60% 감소

### 2. CRON 작업 최적화

**비용 효율적인 모델로 격리된 CRON 작업 구성:**

```json
{
  "name": "Daily Nextcloud Sync",
  "schedule": {"kind": "cron", "expr": "0 3 * * *", "tz": "UTC"},
  "payload": {
    "kind": "agentTurn",
    "message": "Run daily Nextcloud sync",
    "model": "anthropic/claude-haiku-4-5",
    "thinking": "off",
    "timeoutSeconds": 300
  },
  "sessionTarget": "isolated",
  "delivery": {"mode": "none"}
}
```

**주요 최적화 사항:**
- `model: "anthropic/claude-haiku-4-5"` (기본 Sonnet 대신 사용)
- `thinking: "off"` (토큰 최소화)
- `delivery: {mode: "none"}` (오류가 없으면 자동 처리)

**예상 절감액**: CRON 작업에서 Sonnet 대신 Haiku를 사용하면 월 약 $5-10 절감.

### 3. 컨텍스트 관리

**토큰 사용량 최소화:**

**전략 1: 라이트 컨텍스트 모드**
```json
{
  "payload": {
    "kind": "agentTurn",
    "message": "...",
    "lightContext": true
  }
}
```
- 부트스트랩 컨텍스트 감소
- 턴당 2-5K 토큰 절약
- 격리된 작업에 사용 (CRON 작업)

**전략 2: 타겟 메모리 조회**
- `memory_search`를 사용하여 특정 정보 가져오기
- `from` 및 `lines` 파라미터와 함께 `memory_get` 사용
- 전체 MEMORY.md 로딩 방지

**전략 3: 세션 리셋**
```json
{
  "sessionReset": "inactivity (1440 min) + daily (4:00)"
}
```
- 장시간 실행 세션 자동 리셋
- 컨텍스트 비대화 방지
- 매일 새로운 시작

### 4. 무료/로컬 대안

**로컬 추론을 위한 Ollama:**

```bash
# One-time setup
ollama pull qwen2.5:14b
ollama pull llama3.2:3b

# Use for non-critical tasks
{
  "payload": {
    "model": "ollama/qwen2.5:14b",
    "message": "Analyze this log file..."
  }
}
```

**비용**: API 비용 $0 (전기료만 약 $0.50-1/월)

**사용 사례:**
- 로그 분석
- 텍스트 처리
- 코드 리뷰
- 학습 실험

**사용을 피해야 할 경우:**
- 사용자 대면 응답 (품질이 중요)
- 복잡한 추론 (클라우드 모델이 더 우수)
- 시간 민감한 작업 (클라우드가 더 빠름)

---

## 유지 관리 및 모니터링

### 1. 업데이트 전략

**정기적으로 업데이트 확인:**

```bash
# Check current version
openclaw --version

# Check latest release
openclaw update check

# Update when ready
openclaw gateway stop
npm install -g openclaw@latest
openclaw gateway start
```

**업데이트 빈도 권장 사항:**
- **보안 패치**: 즉시 적용
- **기능 릴리스**: 출시 후 1-2주 대기 (다른 사람들이 버그를 찾도록)
- **호환성을 깨는 변경사항**: 별도 인스턴스에서 먼저 테스트

### 2. 백업 전략

**백업 대상:**
- `~/.openclaw/openclaw.json` (구성)
- `~/.openclaw/workspace/` (모든 워크스페이스 파일)
- `~/.openclaw/workspace/memory/` (일일 로그)
- `~/.openclaw/workspace/MEMORY.md` (장기 메모리)

**백업 스크립트:**
```bash
#!/bin/bash
# Backup OpenClaw workspace
tar -czf ~/backups/openclaw-$(date +%Y%m%d).tar.gz \
  ~/.openclaw/openclaw.json \
  ~/.openclaw/workspace/
```

**복원 절차:**
```bash
# Stop OpenClaw
openclaw gateway stop

# Restore files
tar -xzf ~/backups/openclaw-20260420.tar.gz -C ~/

# Restart OpenClaw
openclaw gateway start
```

**백업 빈도:**
- **구성**: 모든 변경 전
- **워크스페이스**: 매일 (자동화)
- **메모리 파일**: 매일 (자동화, 넥스트클라우드 (Nextcloud) 동기화를 통해)

### 3. 상태 모니터링

**상태 확인을 위한 CRON 작업:**

```json
{
  "name": "Weekly System Health Report",
  "schedule": {"kind": "cron", "expr": "0 10 * * 0", "tz": "UTC"},
  "payload": {
    "kind": "agentTurn",
    "message": "Generate weekly system health report: OpenClaw status, API usage, security alerts, CRON job status",
    "model": "anthropic/claude-haiku-4-5"
  },
  "delivery": {
    "mode": "announce",
    "channel": "discord",
    "to": "#personal"
  }
}
```

**모니터링 항목:**
- 오픈클로 (OpenClaw) Gateway 상태 (실행 중/중지됨)
- API 제공자 상태 (속도 제한, 오류)
- CRON 작업 성공/실패율
- 보안 경고 수
- 디스크 공간 사용량
- 메모리 사용량

### 4. 성능 조정

**Gateway 설정:**

```json
{
  "agents": {
    "defaults": {
      "maxConcurrent": 4,
      "subagents": {
        "maxConcurrent": 8
      }
    }
  }
}
```

**다음을 기준으로 조정:**
- **사용 가능한 RAM**: RAM이 많을수록 `maxConcurrent` 값 증가
- **CPU 코어 수**: 코어가 많을수록 `maxConcurrent` 값 증가
- **API 속도 제한**: 제공자 제한 이하로 유지

**예시**: 16 GB RAM 서버
- `maxConcurrent: 4` (주요 세션)
- `subagents.maxConcurrent: 8` (백그라운드 작업)

### 5. 로그 관리

**로그 순환:**

```bash
# Create log rotation config
sudo nano /etc/logrotate.d/openclaw

# Add configuration
/data/.openclaw/workspace/logs/*.log {
    daily
    rotate 30
    compress
    missingok
    notifempty
}
```

**정기적으로 로그 검토:**
```bash
# Check gateway logs
tail -f ~/.openclaw/logs/gateway.log

# Check session logs
tail -f ~/.openclaw/workspace/logs/session-*.log

# Check security logs
tail -f ~/.openclaw/workspace/security-alerts.log
```

---

## 교훈

### 1. 보안 최우선, 항상

**효과가 있었던 것:**
- 컨텍스트 인식 보안 정책 (Control UI vs 디스코드 (Discord))
- 계층형 접근 제어 (허용 목록 + 권한 + 정책)
- 인프라 격리 (금융 서비스 인프라와 AI 분리)
- 감사 로깅 (정책 위반 탐지)

**효과가 없었던 것:**
- 처음에 디스코드 (Discord)를 너무 신뢰함 (정책 없음)
- 소셜 엔지니어링 위험을 예상하지 못함
- 승인된 사용자 = 신뢰할 수 있음으로 가정 (잘못된 생각!)

**교훈**: **제로 트러스트 아키텍처**. 외부 컨텍스트에서 승인된 사용자도 정보를 유출할 수 있습니다 (실수로 또는 침해를 통해).

### 2. 비용 의식이 중요합니다

**효과가 있었던 것:**
- CRON 작업에 Haiku 사용 (40% 비용 절감)
- 비중요 작업에 로컬 Ollama 사용 (무료)
- 격리된 작업에 라이트 컨텍스트 모드 사용
- 자동화를 위한 사고 모드 OFF

**효과가 없었던 것:**
- 처음에 모든 작업에 Sonnet 사용 (비용이 많이 듦)
- API 사용량을 정기적으로 모니터링하지 않음
- CRON 작업 최적화를 잊음

**교훈**: **모델을 적절히 선택하세요**. 대부분의 작업에는 프리미엄 모델이 필요하지 않습니다.

### 3. 문서화가 중요합니다

**효과가 있었던 것:**
- AGENTS.md (재시작 후에도 지침 유지)
- MEMORY.md (장기 메모리 지속성)
- 일일 로그 (감사 추적 + 디버깅)
- IDEAS.md (프로젝트 추적)

**효과가 없었던 것:**
- "머릿속 메모"에 의존 (재시작 후 손실)
- 오픈클로 (OpenClaw)가 기억한다고 가정 (그렇지 않음)
- 구두 지침 (서면 정책이 필요)

**교훈**: **기록되지 않은 것은 존재하지 않습니다.** 오픈클로 (OpenClaw)는 파일 없이는 지속적인 메모리가 없습니다.

### 4. 단순하게 시작하고, 점차 확장하세요

**효과가 있었던 것:**
- Control UI만으로 시작 (로컬, 보안)
- 디스코드 (Discord) 추가 전 철저한 테스트
- 채널을 하나씩 추가
- 점진적으로 보안 정책 구현

**효과가 없었던 것:**
- 한 번에 모든 것을 구성하려 함
- 이해하기 전에 기능 활성화
- 기본값이 안전하다고 가정 (그렇지 않음)

**교훈**: **기어가고, 걷고, 뛰세요**. 복잡성을 추가하기 전에 기본을 마스터하세요.

### 5. 모니터링하고, 측정하고, 개선하세요

**효과가 있었던 것:**
- 주기적 확인을 위한 CRON 작업 (깃허브 (GitHub) 이슈, 릴리스, 보안)
- 비용 추적 (월간 API 지출)
- 보안 감사 로그 (정책 위반 탐지)
- 세션 상태 확인 (모델 사용량, 토큰 수)

**효과가 없었던 것:**
- 처음에 비용을 추적하지 않음 (예상치 못한 청구서)
- 보안 모니터링 없음 (정책 위반 놓침)
- 사후 대응이 아닌 사전 대응 부재

**교훈**: **측정하지 않는 것은 개선할 수 없습니다.** 첫날부터 모니터링을 설정하세요.

### 6. 커뮤니티 및 지원

**효과가 있었던 것:**
- 오픈클로 (OpenClaw) 문서를 철저히 읽음
- 명확한 질문하기
- 안전한 환경에서 테스트
- 이슈 문서화 (깃허브 (GitHub) 이슈 #61294)

**효과가 없었던 것:**
- 기능이 작동한다고 가정 (디스코드 (Discord) 음성 기능 오류)
- 알려진 이슈를 먼저 확인하지 않음
- 릴리스 노트 건너뜀

**교훈**: **먼저 매뉴얼을 읽고, 그 다음 릴리스 노트를 읽으세요 (RTFM, then RTFR).** 버그라고 가정하기 전에 문서와 이슈를 확인하세요.

---

## 결론

### 핵심 요점

1. **보안은 목적지가 아닌 여정입니다**: 지속적으로 평가하고 개선하세요
2. **컨텍스트가 중요합니다**: Control UI ≠ 디스코드 (Discord) ≠ 공개 인터넷
3. **비용 최적화가 가능합니다**: 스마트한 모델 선택으로 40-60% 절감 가능
4. **문서화는 시간을 절약합니다**: 기록해두면 미래의 자신이 감사할 것입니다
5. **제로 트러스트 아키텍처**: 신뢰하되 검증하세요, 승인된 사용자도 마찬가지
6. **작게 시작하고, 반복하세요**: 한 번에 모든 것을 하려 하지 마세요

### 권장 다음 단계

**이 가이드를 따른 후:**

1. **철저히 테스트하세요**:
   - 모든 채널이 올바르게 응답하는지 확인
   - 보안 정책이 적용되는지 확인
   - 비용 추적이 작동하는지 확인

2. **구체적인 내용을 문서화하세요**:
   - 문서에 채널 ID 추가
   - 위협 모델 문서화
   - 아키텍처 결정 기록

3. **모니터링 설정:**
   - 상태 확인을 위한 CRON 작업
   - 보안 감사 로깅
   - API 비용 추적

4. **인시던트에 대비하세요**:
   - 인시던트 대응 계획 작성
   - 백업/복원 절차 테스트
   - 에스컬레이션 경로 문서화

5. **최신 상태 유지:**
   - 오픈클로 (OpenClaw) 릴리스 모니터링
   - 깃허브 (GitHub) 이슈 확인
   - 커뮤니티 토론 참여

### 리소스

- **공식 문서**: https://docs.openclaw.ai
- **깃허브 (GitHub)**: https://github.com/openclaw/openclaw
- **커뮤니티 디스코드 (Discord)**: https://discord.com/invite/clawd
- **보안 모범 사례**: https://docs.openclaw.ai/security
- **스킬 마켓플레이스**: https://clawhub.ai

---

## 부록 A: 빠른 참조 명령어

**Gateway 관리:**
```bash
openclaw gateway start        # Start gateway
openclaw gateway stop         # Stop gateway
openclaw gateway restart      # Restart gateway
openclaw status               # Check status
```

**구성:**
```bash
openclaw config view          # View current config
openclaw config edit          # Edit config file
openclaw config validate      # Validate config
```

**업데이트:**
```bash
openclaw update check         # Check for updates
openclaw update run           # Update OpenClaw
```

**디버깅:**
```bash
openclaw logs gateway         # View gateway logs
openclaw logs session         # View session logs
openclaw doctor               # Run diagnostics
```

---

## 부록 B: 구성 파일 템플릿

**최소 보안 디스코드 (Discord) 구성:**

```json
{
  "channels": {
    "discord": {
      "enabled": true,
      "token": "***",
      "groupPolicy": "allowlist",
      "dmPolicy": "pairing",
      "allowFrom": ["YOUR_USER_ID"],
      "guilds": {
        "YOUR_GUILD_ID": {
          "requireMention": false,
          "users": ["YOUR_USER_ID"]
        }
      }
    }
  }
}
```

**비용 최적화된 CRON 작업:**

```json
{
  "name": "Example CRON Job",
  "schedule": {"kind": "cron", "expr": "0 * * * *", "tz": "UTC"},
  "payload": {
    "kind": "agentTurn",
    "message": "Task description here",
    "model": "anthropic/claude-haiku-4-5",
    "thinking": "off",
    "lightContext": true
  },
  "sessionTarget": "isolated",
  "delivery": {"mode": "none"}
}
```

---

## 부록 C: 보안 체크리스트

**배포 전:**
- [ ] 위협 모델 정의됨
- [ ] 리소스 요구사항 충족됨
- [ ] API 키 보안 처리됨
- [ ] 백업 전략 수립됨

**초기 구성:**
- [ ] Gateway 인증 활성화됨 (강력한 토큰)
- [ ] Control UI가 로컬호스트로 제한됨
- [ ] 상승된 권한 명령이 Control UI로 제한됨
- [ ] 기본 모델 선택됨

**디스코드 (Discord) 설정:**
- [ ] 비공개 서버 생성됨
- [ ] 봇 권한 최소화됨 (읽기/전송만)
- [ ] 사용자 허용 목록 구성됨
- [ ] 길드별 사용자 목록 설정됨
- [ ] 채널 격리 구현됨

**보안 정책:**
- [ ] AGENTS.md에 인프라 보안 정책 포함됨
- [ ] 민감한 파일 접근 제한 문서화됨
- [ ] 명령어 블랙리스트 구현됨
- [ ] 컨텍스트 감지 작동 확인 (Control UI vs 디스코드 (Discord))
- [ ] 감사 로깅 활성화됨

**모니터링:**
- [ ] 상태 확인 CRON 작업 구성됨
- [ ] 보안 감사 CRON 작업 구성됨
- [ ] 비용 추적 활성화됨
- [ ] 로그 순환 구성됨

**인시던트 대응:**
- [ ] 인시던트 대응 계획 문서화됨
- [ ] 백업/복원 테스트 완료됨
- [ ] API 키 교체 절차 문서화됨
- [ ] 긴급 종료 절차 문서화됨

---

**문서 버전:** 1.0  
**최종 업데이트:** 2026-04-20  
**작성자:** 실제 배포 경험 기반  
**라이선스:** CC BY-SA 4.0 (출처 표기 후 공유 및 수정 가능)

---

*이 가이드는 교육 목적으로 있는 그대로 제공됩니다. 항상 본인의 구체적인 위협 모델 및 사용 사례에 맞게 보안 정책을 검토하고 조정하세요. 의심스러울 때는 더 제한적인 설정을 기본값으로 하세요.*

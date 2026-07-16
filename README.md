# n8n + Ollama Local Chatbot

API 키와 토큰 과금 없이 로컬에서 실행하는 AI 챗봇 예제입니다. n8n이 채팅 워크플로를 관리하고, Ollama의 `qwen3:4b-instruct` 모델이 답변을 생성합니다.

## 구성

```text
n8n Chat Trigger
        ↓
    AI Agent
        ↑
Ollama Chat Model
```

- **n8n**: 채팅 입력과 AI 워크플로 관리
- **Ollama**: 로컬 LLM 서버
- **qwen3:4b-instruct**: 내부 추론을 길게 출력하지 않는 채팅용 모델
- **Docker Compose**: 서비스와 데이터 볼륨 관리

## 요구 사항

- Docker Desktop 또는 Docker Engine
- Docker Compose
- 모델 저장 공간 약 3GB 이상

## 시작하기

```bash
docker compose up -d
```

처음 실행하면 `ollama-model` 서비스가 `qwen3:4b-instruct`를 자동으로 내려받습니다. 다운로드 상태는 다음 명령으로 확인할 수 있습니다.

```bash
docker compose logs -f ollama-model
```

다운로드가 끝난 `ollama-model` 컨테이너의 상태가 `Exited (0)`으로 표시되는 것은 정상입니다. 모델을 준비하는 일회성 초기화 컨테이너이며, 실제 추론은 계속 실행 중인 `ollama` 컨테이너가 담당합니다.

모델 설치 여부를 확인합니다.

```bash
docker compose exec ollama ollama list
```

n8n은 다음 주소에서 열 수 있습니다.

```text
http://localhost:5678
```

## n8n 워크플로 설정

다음 노드를 추가하고 연결합니다.

1. `When chat message received`
2. `AI Agent`
3. `Ollama Chat Model`

일반 연결:

```text
When chat message received → AI Agent
```

AI 모델 연결:

```text
Ollama Chat Model → AI Agent의 Chat Model 포트
```

### Ollama Credential

Ollama Credential의 Base URL을 다음과 같이 설정합니다.

```text
http://ollama:11434
```

API Key는 입력하지 않습니다. Docker 컨테이너 내부에서 `localhost`는 n8n 컨테이너 자신을 가리키므로 `http://localhost:11434`를 사용하면 연결되지 않습니다.

### 모델

Ollama Chat Model 노드에서 다음 모델을 선택합니다.

```text
qwen3:4b-instruct
```

n8n의 `Open chat` 버튼을 누르면 별도 웹 화면 없이 챗봇을 테스트할 수 있습니다.

## 상태 확인

```bash
docker compose ps
docker compose exec ollama ollama ps
docker compose logs -f ollama
docker compose logs -f n8n
```

Ollama API를 직접 테스트하려면 다음 요청을 사용합니다.

```bash
curl http://localhost:11434/api/chat \
  -H 'Content-Type: application/json' \
  -d '{
    "model": "qwen3:4b-instruct",
    "messages": [
      {"role": "user", "content": "안녕! 한 문장으로 인사해줘."}
    ],
    "stream": false
  }'
```

## 문제 해결

### Error fetching options from Ollama Chat Model

Credential의 Base URL을 확인합니다.

```text
http://localhost:11434  ❌
http://ollama:11434     ✅
```

수정 후 Credential에 `Connection tested successfully`가 표시되는지 확인하고, Ollama Chat Model 노드를 닫았다가 다시 엽니다.

### 질문을 보냈지만 답변이 늦는 경우

일반 `qwen3:4b`는 Thinking 모델이라 CPU 환경에서 최종 답변 전에 긴 내부 추론을 생성할 수 있습니다. 이 프로젝트는 빠른 채팅을 위해 비추론 모델인 `qwen3:4b-instruct`를 사용합니다.

### macOS Docker Desktop에서 속도가 느린 경우

macOS Docker Desktop의 Ollama 컨테이너는 Apple GPU의 Metal 가속을 사용하지 못할 수 있어 CPU로 실행됩니다. 더 빠른 속도가 필요하면 Ollama를 macOS에 직접 설치하고, n8n Credential을 다음 주소로 변경할 수 있습니다.

```text
http://host.docker.internal:11434
```

## 종료 및 삭제

컨테이너를 종료합니다.

```bash
docker compose down
```

n8n 데이터와 Ollama 모델은 Docker 볼륨에 유지됩니다. 볼륨까지 모두 삭제하려면 다음 명령을 사용합니다.

```bash
docker compose down -v
```

> 위 명령은 n8n 설정과 다운로드한 모델을 모두 삭제합니다.

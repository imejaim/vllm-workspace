# vLLM + Qwen3.5-35B-A3B 재실행 기록 및 체크리스트

작성일 기준: 2026-03-27  
환경: Ubuntu 22.04.5 LTS / NVIDIA RTX PRO 6000 Blackwell x2 / Python 3.10.12

---

## 1. 현재 확인된 정상 상태

다음 항목이 확인됨.

- `~/vllmvenv` 가상환경 사용 가능
- `vllm 0.18.0` 설치됨
- `torch 2.10.0` 설치됨
- 모델 폴더 `~/models/Qwen3.5-35B-A3B` 존재
- Qwen3.5 모델 shard와 tokenizer/config 파일 존재
- `vllm serve` 는 `--enforce-eager` 옵션에서 정상 기동
- `/v1/models` 정상 응답
- `/v1/chat/completions` 정상 응답
- `enable_thinking=false` 설정 시 `VLLM_OK` 정상 응답

---

## 2. 서버/환경 점검 명령

```bash
echo "=== OS ==="
uname -a
cat /etc/os-release

echo
echo "=== GPU / DRIVER ==="
nvidia-smi

echo
echo "=== Python ==="
python3 --version
which python3

echo
echo "=== pip / venv ==="
python3 -m pip --version
python3 -m venv --help >/dev/null && echo "venv OK" || echo "venv NG"

echo
echo "=== Disk ==="
df -h ~
```

정상 기준:
- Ubuntu 22.04.5
- NVIDIA Driver 580.126.09
- CUDA Version 13.0 표시
- Python 3.10.12
- `venv OK`
- 디스크 여유 충분

---

## 3. 현재 사용하는 경로

- 가상환경: `~/vllmvenv`
- 모델 폴더: `~/models/Qwen3.5-35B-A3B`
- 작업 폴더: `~/project/101_vllm_setup`

---

## 4. 가상환경 및 설치 상태 확인

```bash
source ~/vllmvenv/bin/activate

python --version
which python
python -m pip show vllm torch | egrep "Name:|Version:|Location:"
python -m pip show transformers | egrep "Name:|Version:|Location:"
```

확인된 예시 상태:
- `vllm 0.18.0`
- `torch 2.10.0`
- `transformers`는 별도 점검 필요

---

## 5. 모델 파일 완전성 확인

```bash
source ~/vllmvenv/bin/activate

echo "=== model tail ==="
ls ~/models/Qwen3.5-35B-A3B | tail

echo
echo "=== must-have files check ==="
test -f ~/models/Qwen3.5-35B-A3B/config.json && echo "config.json OK" || echo "config.json MISSING"
test -f ~/models/Qwen3.5-35B-A3B/tokenizer.json && echo "tokenizer.json OK" || echo "tokenizer.json MISSING"
test -f ~/models/Qwen3.5-35B-A3B/tokenizer_config.json && echo "tokenizer_config.json OK" || echo "tokenizer_config.json MISSING"
test -f ~/models/Qwen3.5-35B-A3B/model.safetensors.index.json && echo "index OK" || echo "index MISSING"
test -f ~/models/Qwen3.5-35B-A3B/model.safetensors-00014-of-00014.safetensors && echo "last shard OK" || echo "last shard MISSING"
```

정상 확인됨:
- `config.json OK`
- `tokenizer.json OK`
- `tokenizer_config.json OK`
- `index OK`
- `last shard OK`

---

## 6. vLLM 실행 시도 중 겪은 문제

### 6-1. transformers import 오류
초기에는 `vllm 0.18.0` 실행 시 아래와 같은 import 오류가 발생함.

- `ALLOWED_ATTENTION_LAYER_TYPES`
- `ALLOWED_LAYER_TYPES`

이건 `vllm`과 `transformers` 버전 궁합 문제로 판단.

---

### 6-2. compile / FakeTensorMode 오류
이후 실행 시 아래 계열 오류가 발생함.

- `FakeTensorMode`
- `standalone_compile`
- `PiecewiseBackend`
- `aot_compile`
- `WorkerProc hit an exception`

즉, `vllm 0.18.0`의 compile 경로와 현재 PyTorch 조합에서 충돌이 발생한 것으로 판단.

---

## 7. 최종적으로 성공한 실행 방법

핵심: `--enforce-eager`

```bash
source ~/vllmvenv/bin/activate
export HF_HUB_OFFLINE=1

vllm serve ~/models/Qwen3.5-35B-A3B \
  --host 0.0.0.0 \
  --port 8000 \
  --tensor-parallel-size 2 \
  --max-model-len 32768 \
  --served-model-name Qwen3.5-35B-A3B \
  --reasoning-parser qwen3 \
  --enable-auto-tool-choice \
  --tool-call-parser qwen3_coder \
  --enforce-eager
```

이 조합에서 서버가 정상적으로 살아남고 API 응답까지 확인됨.

---

## 8. 서버 생존 확인

```bash
ps aux | grep vllm
```

정상 예시:
- `vllm serve ... --enforce-eager` 프로세스가 살아 있음

---

## 9. API 확인 명령

### 9-1. health
```bash
curl http://127.0.0.1:8000/health
```

### 9-2. 모델 목록
```bash
curl http://127.0.0.1:8000/v1/models
```

정상 응답 확인:
- `Qwen3.5-35B-A3B` 표시됨

---

## 10. 채팅 응답 확인

`python` 명령이 없는 쉘에서는 `python3` 사용.

```bash
curl -s -X POST http://127.0.0.1:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer dummy" \
  -d '{
    "model":"Qwen3.5-35B-A3B",
    "messages":[
      {"role":"system","content":"You are a concise coding assistant."},
      {"role":"user","content":"Reply with exactly: VLLM_OK"}
    ],
    "temperature":0,
    "max_tokens":20,
    "chat_template_kwargs":{"enable_thinking":false}
  }' | python3 -m json.tool
```

정상 응답 확인:

```json
"content": "VLLM_OK"
```

---

## 11. 현재 의미

지금 기준으로 확인된 것은 다음과 같음.

- vLLM 설치 및 실행 가능
- 로컬 모델 경로에서 정상 서빙 가능
- OpenAI 호환 API 정상 동작
- Qwen3.5-35B-A3B를 Blackwell 2GPU에서 최소 안정 상태로 운용 가능
- 현재 가장 안전한 실행 기준은 `--enforce-eager`

---

## 12. 다음 단계

다음으로 진행할 수 있는 일:

1. `~/.config/opencode/opencode.json` 확인
2. `opencode models` 실행
3. `opencode`에서 local vLLM 연결 확인
4. 필요 시 실행 명령을 `run_vllm.sh`로 저장
5. 필요 시 systemd 서비스화

---

## 13. 추천 실행 스크립트 예시

파일명: `~/project/101_vllm_setup/run_vllm_qwen35.sh`

```bash
#!/usr/bin/env bash
set -e

source ~/vllmvenv/bin/activate
export HF_HUB_OFFLINE=1

vllm serve ~/models/Qwen3.5-35B-A3B \
  --host 0.0.0.0 \
  --port 8000 \
  --tensor-parallel-size 2 \
  --max-model-len 32768 \
  --served-model-name Qwen3.5-35B-A3B \
  --reasoning-parser qwen3 \
  --enable-auto-tool-choice \
  --tool-call-parser qwen3_coder \
  --enforce-eager
```

실행 권한 부여:

```bash
chmod +x ~/project/101_vllm_setup/run_vllm_qwen35.sh
```

실행:

```bash
~/project/101_vllm_setup/run_vllm_qwen35.sh
```

---

## 14. 한 줄 요약

현재 성공한 기준은 아래 한 줄이다.

```bash
source ~/vllmvenv/bin/activate && export HF_HUB_OFFLINE=1 && vllm serve ~/models/Qwen3.5-35B-A3B --host 0.0.0.0 --port 8000 --tensor-parallel-size 2 --max-model-len 32768 --served-model-name Qwen3.5-35B-A3B --reasoning-parser qwen3 --enable-auto-tool-choice --tool-call-parser qwen3_coder --enforce-eager
```

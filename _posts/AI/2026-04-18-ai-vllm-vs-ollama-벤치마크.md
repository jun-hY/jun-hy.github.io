---
layout: post
title: "[AI] vLLM vs Ollama 벤치마크"
date: 2026-04-18 21:40 +0900
category: ['AI']
image: /assets/img/post/ai-bench/image_logo.png
tag: ['AI', 'benchmark', 'LLM']
---

[vLLM](https://vllm.ai/) 과 [Ollama](https://ollama.com/) 의 추론 속도 벤치마킹

## 1. 개요

`MoE` 모델이 출시되면서부터 메모리 부족 문제가 더욱이 부각되고 있다.

`vLLM` 는 이를 해결하기 위해 `PagedAttention` 이라는 기법을 이용해 메모리를 관리하는 LLM 구동 인터페이스이다.

그래서, `Ollama` 와 같은 모델을 구동 중일때 컨텍스트 토큰의 크기에 따라 얼마나 차이나는 지 테스트해보고 싶어 벤치마크를 진행해보게 되었다.

## 2. 벤치마크 코드

유튜버 [괴발자](https://www.youtube.com/@weirdeveloper) 님의 [벤치마크 코드](https://github.com/weirdeveloper/llm-inference-benchmark/)를 가져왔다.

벤치마크 코드에 대해 간략히 소개하자면

1. `tiktoken` 을 이용한 정확한 토큰 길이 계산 -> 목표 토큰 수에 정확하게 맞는 프롬프트를 동적으로 생성
2. `kv-cache` 무력화를 위한 난수 주입         -> `kv-cache` 적중률을 낮춰 순수 연산에 대한 지표 획득을 목표로 함
3. 더미 요청을 통한 `warm-up` 루프가 포함     -> 메모리에 모델 가중치를 완전히 적재시킴


## 3. 벤치마크 결과

`vLLM` 은 `Ollama` 의 메모리 동적 점유와 다르게 메모리를 예약하는 방식으로 작동하기 때문에 별도의 설정없이는 그래픽카드의 Vram 이 부족할 수 있다.

그래서, `qwen3.5-0.8b` 모델을 사용하여 벤치마크를 진행했다.

`Ollama` 는 라이브러리에서 `qwen3.5-0.8b` 모델을 다운받아 바로 진행했고, `vLLM` 서버를 올린 커맨드는 다음과 같다.

```sh
vllm serve "Qwen/Qwen3.5-0.8B" \
    --api-key vLLM_test \
    --gpu-memory-utilization 0.85 \
    --max-model-len 16384 \
    --port 30000
```

vram 이 부족하여 최대 컨텍스트 길이를 `16384` 로 설정하여 진행했다.

![](/assets/img/post/ai-bench/image_result_ollama.png)
_Ollama 벤치마크 결과_

![](/assets/img/post/ai-bench/image_result_vllm.png)
_vLLM 벤치마크 결과_


## 4. 후기

`vLLM` 이 `--gpu-memory-utilization`, `--max-model-len` 등등 수동으로 옵션을 조절해야해서 일반 사용자에게 불편함이 있지만, 여러 옵션을 테스트해서 본인 사양에 맞출 수만 있다면 `Ollama` 보다 빠른 추론 속도를 보여주기 때문에 (offload 가 된 경우 제외) 토이 프로젝트용으로 사용해볼만 한 것 같다.

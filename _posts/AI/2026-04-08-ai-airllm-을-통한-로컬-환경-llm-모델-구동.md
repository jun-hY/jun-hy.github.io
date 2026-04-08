---
layout: post
title: "[AI] AirLLM 을 통한 로컬 환경 LLM 모델 구동"
date: 2026-04-08 23:40 +0900
category: ['AI']
image: /assets/img/post/airllm/image1.png
tag: ['AI', 'Quantization', 'LLM']
---

> 99.39 s...?????

![](/assets/img/post/airllm/image0.png)
_[AirLLM](https://github.com/lyogavin/airllm)_

최근, AI 와 연관된 github 레포지토리를 탐색하다가 `AirLLM` 이라는 프로젝트를 발견했다. Readme 를 읽어보니 대강 LLM interface memory 를 최적화해주는 그런 프로젝트인 것 같았는데...

![](/assets/img/post/airllm/image2.png)
_무려 405B 파라미터를 8GB Vram으로 돌리게 해준다는..._

405B 파라미터의 LLama3.1 모델을 8GB Vram 으로 돌리게 해준다는 문구를 보고 호기심에 테스트를 진행해보게 되었다.

테스트용으로 선정된 모델은 [Mistral 7B Instruct v0.1](https://huggingface.co/mistralai/Mistral-7B-Instruct-v0.1) 모델. `AirLLM` 이 지원하는 모델 중 가장 친숙한 모델이라서 고르게 되었다.

테스트를 진행해보고 속도면에서 사용 가능한 수준이면 같은 파라미터 수준의 더 성능이 좋은 모델을 찾아서 `langchain` 과 함께 모델을 서빙하는 프로젝트를 진행해볼 생각이였다.

## 1. 환경 구성

의존성이 겹치는 것을 막기위해 python 가상환경을 만들어 진행하자.

```sh
python -m venv .venv

# window 에서 가상환경 활성화
.venv/Scripts/Activate.ps1

# Linux 에서 가상환경 활성화
source .venv/bin/activate
```

`torch` 는 `AirLLM`을 설치하면 같이 설치되지만, 기본적으로 pip 를 통해 제공되는 `pytorch`는 cpu용 이기 때문에 gpu 연산을 위해 gpu 사용이 가능한 `torch`를 설치하기 위해 아래의 명령어를 입력해야한다.

```sh
# 높은 버전이 하위 버전을 지원하기 때문에 cu128, cu130 으로 본인 cuda 버전에 맞게 해도 된다.
pip install torch torchvision --index-url https://download.pytorch.org/whl/cu126 
```

현재, `AirLLM` 프로젝트는 의존성이 박살난 상태다. 나도 알고 싶지 않았다.

구글을 뒤져 나와 같은 경험을 한 선배님들의 글을 찾아보며 의존성을 박살난 원인을 찾았다.

`transformers` 와 `optimum`은 아래와 같은 버전으로 설치하자.

```sh
pip install transformers==4.48.0 optimum==1.17.0 airllm
```

위 버전을 설치하지 않으면 아래 2개의 에러를 맞이하게 될 것이다....

```sh
ModuleNotFoundError: No module named 'optimum.bettertransformer'
```


```sh
  File "airllm-local\.venv\Lib\site-packages\transformers\models\mistral\modeling_mistral.py", line 165, in forward
    cos, sin = position_embeddings
    ^^^^^^^^
TypeError: cannot unpack non-iterable NoneType object
```

양자화를 원한다면 `BitsAndBytes` 도 같이 설치해주자.

```sh
pip install bitsandbytes
```

여기까지가 환경 구성은 끝이다.

## 2. 테스트 진행

테스트는 간단하게 Readme에 적힌 코드를 복사하여 타이머 코드만 추가하였다.

```py
import time
start = time.time()

"""
모델 로직
"""

print(f"{time.time() - start:5} s")
```

테스트를 해보면

![](/assets/img/post/airllm/image1.png)

그렇다. 이 포스트의 썸네일에서 보았듯 99.39 초라는 어마어마하게 느린, 심지어 컨텍스트 크기도 output 기준 20토큰이다.

필자는 `4bit` 양자화를 적용했는데 `8bit` 기준 150초 내외, 컨텍스트를 압축하지않은 설정 기준 300초 라는 수치가 나왔다.

필자의 컴퓨터 사양이 부족한게 아니냐고 할 수 있는데 모델을 구동하는 데 필요한 부분만 나열해 보자면

| 종류 | 제품 |
|----|----|
| CPU | AMD R7 7800x3d |
| GPU | NVIDIA RTX 5060ti 8GB |
| RAM | Oloy DDR5 32GB |

로 물론 Vram 이 부족한 것은 맞지만, 연산 속도가 느리냐고 하면 이렇게까지 느릴 정도는.... 아니라고 생각한다.

(아 그래픽 카드 바꾸고 싶다.)

## 3. 원인 분석

우선, `AirLLM` 의 동작 원리를 보면 트랜스포머 모델의 각 레이어를 모두 Vram에 올려 병렬로 처리하는 기존 방식과 달리 레이어를 하나씩 올려 계산하는 방식을 취하고 있다.

그래서 기존 처리 방식에 비해서 느릴 수 밖에 없고, 특히나 파라미터가 많은 모델로 갈 수록 기존 처리 방식에 비해 더 느려질 것으로 예상이 된다.

## 4. 후기

이 프로젝트를 이용해 LLM 서빙 프로젝트를 한번 더 진행해보면서 `Langchain`에 대해서 더 알아보려고 했는데 완전히 실패해버렸다...

그래도, 비슷한 llm 압축 프로젝트를 찾아보며 `turboquant` 에 대해서 알게 되었고 공식 오픈소스 릴리즈는 없지만 커뮤니티 구현체를 이용하여 `vLLM` 엔진을 써볼 겸 4bit 양자화에 `turboquant` 를 적용해서 다시 시도해보면 괜찮을 것 같다.
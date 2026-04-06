---
layout: post
title: "[AI] Colab 무료 요금제로 LLM 모델 서빙해보기"
date: 2026-04-06 18:06 +0900
category: ['AI']
image: /assets/img/post/colab_ai/image0.png
tag: ['Web', 'AI', 'Colab', 'Quantization', 'sLLM', 'LLM']
---

## 1. 프로젝트 개요

### 1-1. 선정 배경

> [MicroSoft Phi-4 모델](https://huggingface.co/microsoft/Phi-4) 을 활용한 버라이어티 Colab 모델 서빙 프로?젝트 후기.

작년 2학기 중반 즈음, 25년 9월부터 11월 사이 였던 것같다. 딱히 학교 과제도 없고 AI 관련 프로젝트를 진행해보고 싶은 욕심에 찾아보다가 MS Phi-4 모델을 찾게 되었다.

AI 관련 프로젝트를 구상할 때 마다 항상 GPU 가 진입장벽처럼 느껴졌었는데, Phi-4 모델은 mini 모델로 구동 시에 **3.8B** 파라미터를 사용하여 실제 VRAM 사용량이 8GB(4bit 양자화시 약 3GB) 수준으로 굉장히 낮은 사용량을 보여줬기 때문에 프로젝트를 시작하기로 마음 먹을 수 있었던 것 같다.

사실 파라미터 수에서 알 수 있듯, Phi-4 모델은 LLM 이라기보단 sLLM 이다. 메인 서비스 용도보단 Agent 용도로 많이 사용되는 것 같고, 실제로 기본 모델의 경우 수학 능력이 GPT-4o-mini 보다 조금 좋게 측정된 것을 볼 수 있다. mini 모델의 경우 o1-mini 모델과 비교하는데 수학 능력의 경우 더 높게 측정되는 것 같다.

그리고, Colab 무료 요금제에서 사용할 수 있는 런타임 중 무려 **Tesla T4** GPU 를 사용할 수 있다. 물론 Tesla 아키텍쳐의 구형 GPU 에 12GB VRAM 밖에 안되지만, Phi-4 모델을 구동하기엔 충분하기에 이번 프로젝트는 **Colab 무료 요금제로 (s)LLM 서빙하기** 되시겠다.

해당 프로젝트는 LLM 서빙에 대한 경험도 없고 사전지식도 없어 Gemini 모델을 사용해서 진행했다.

### 1-2. 사용 스택

|스택|이유|
|----|----|
| python | tranformer 사용을 위함 |
| langchain & streamlit | web gui 제공 및 채팅 ui 제공함, 채팅 ui 와 모델을 연동하기 위함 |
| Ngrok | 외부 엔드포인트 제공 |


## 2. 프로젝트 진행

### 2-1. 개발 진행

COSS 사업단에서 진행한 Co-Week 에서 진행한 랭체인을 활용한 AI 챗봇 구현 강의에서 교수님께서 제공해주신 langchain 템플릿을 사용했다.

기본 템플릿에는 gemini api를 사용하기 때문에 모델을 로드하는 코드가 없다. 

그 부분을 추가했다.

또한, 양자화를 진행해서 파라미터를 압축해보았다. Tesla T4 환경에서 필요없는 조치긴 하지만, 이후 로컬 환경에서 모델을 돌릴 수도 있을지 모르니 미리 연습하는 느낌으로 적용해보았다.

모델 레퍼.
```py
class Phi4HuggingFace(LLM):
    model: Any
    processor: Any
    generation_config: GenerationConfig

    @property
    def _llm_type(self) -> str:
        return "phi4-huggingface"

    def _call(self, prompt: str, stop: Optional[List[str]] = None, **_kwargs: Any) -> str:
        inputs = self.processor(prompt, images=None, return_tensors='pt').to(self.model.device)
        input_token_len = inputs['input_ids'].shape[1]

        generate_ids = self.model.generate(
            **inputs,
            generation_config=self.generation_config,
        )

        response_ids = generate_ids[:, input_token_len:]
        response = self.processor.batch_decode(
            response_ids, skip_special_tokens=True, clean_up_tokenization_spaces=False
        )[0]

        if response.endswith('<|end|>'):
            response = response[:-len('<|end|>')].strip()

        return response

    @property
    def _identifying_params(self) -> Dict[str, Any]:
        return {"model_path": MODEL_PATH, "quantization": "4-bit"}
```

모델 로딩.

`Tesla T4` 모델은 `bfloat16` 를 지원하지 않는다는 AI 의 대답에 `float16` 타입으로 바꿨다.
```py
@st.cache_resource
def load_model() -> Phi4HuggingFace:
    processor = AutoProcessor.from_pretrained(MODEL_PATH, trust_remote_code=True)

    quantization_config = BitsAndBytesConfig(
        load_in_4bit=True,
        bnb_4bit_quant_type="nf4",
        bnb_4bit_compute_dtype=torch.float16,
        bnb_4bit_use_double_quant=True,
    )

    model = AutoModelForCausalLM.from_pretrained(
        MODEL_PATH,
        trust_remote_code=True,
        device_map='auto',
        quantization_config=quantization_config,
        _attn_implementation='sdpa',
    )

    generation_config = GenerationConfig.from_pretrained(MODEL_PATH, 'generation_config.json')
    generation_config.max_new_tokens = 20000

    return Phi4HuggingFace(model=model, processor=processor, generation_config=generation_config)
```

개인용 LLM 을 기준으로 진행하였기 때문에 RAG(검색 증강 생성)을 적용해 보았다.

RAG를 적용하기 위해 Tavily를 활용하였다.

```py
TOOLS = {
    "name": "tavily_search",
    "description": "Performs a web search using the Tavily API to find up-to-date information on a given topic.",
    "parameters": {
        "query": {
            "description": "The search query or topic to look up.",
            "type": "str"
        }
    }
}
```

prompt에 tools을 추가하여 endpoint에 요청하는 방식으로 작성하였다.

프롬프트의 템플릿은 [Phi cookbook](https://github.com/microsoft/PhiCookBook?tab=readme-ov-file) 을 활용하여 아래와 같이 작성되었다.

```py
RAG_TEMPLATE = """<|system|>
{system_message}
<|tool|>
[{tools}]
<|/tool|>
<|end|>
<|user|>
[검색 결과] :
{context}
[유저 메세지] :
{user_message}
<|end|>
<|assistant|>"""
```

채팅 로직은 아래와 같이 구현되었다.

```py
def ask_llm(user_message: str) -> str:
    return rag_chain.invoke({
        "system_message": "Your name is Phi, an AI math expert developed by Microsoft.",
        "user_message": user_message,
        "tools": TOOLS
    })

for msg in st.session_state.messages:
    with st.chat_message(msg["role"]):
        st.write(msg["content"])

if query := st.chat_input("질문을 입력하세요."):
    st.session_state.messages.append({"role": "user", "content": query})
    st.chat_message("user").write(query)
    response = ask_llm(query)
    st.session_state.messages.append({"role": "assistant", "content": response})
    st.chat_message("assistant").write(response)
```


이제 `Streamlit` 을 사용하여 모델을 서빙하는 부분을 작성할 차례이다.

기본적으로 `colab` 환경을 기반으로 제작되었기 때문에 위 모델 & 채팅 로직이 포함된 파일이 하나 작성되고 `subprocess` 를 활용하여 `Streamlit` 으로 `python` 스크립트를 백그라운드에서 동작하는 코드를 작성했다.

```py
from pyngrok import ngrok
import os
import subprocess

PORT = 8501

# Streamlit 앱을 백그라운드에서 실행(포트: 8501)
process = subprocess.Popen(["streamlit", "run", "app.py", "--server.port", str(PORT), "--server.enableCORS", "true", "--server.enableXsrfProtection", "false"], stdout=subprocess.PIPE, stderr=subprocess.PIPE)

# ngrok 인증 토큰 설정(ngrok 홈에서 계정생성 후, 받을 수 있음)
ngrok.set_auth_token(os.environ['NGROK_TOKEN'])

# ngrok 터널 생성
public_url = ngrok.connect(PORT, bind_tls=True)
print(f"Streamlit 앱 접속 링크: {public_url}")
```

외부 접속을 상정하여 `Ngrok` 을 통해 endpoint 연결했다.

## 3. 프로젝트 성과

### 3-1. 기술적 성과

이번 프로젝트는 AI에게 개발을 거의 맡긴 부분도 있고, 강의 자료를 활용하여 작성되었기 때문에 내가 무엇인가를 이루었다기 보다 LLM 모델을 서빙할 때 이런 것이 필요하구나 정도를 알아간 프로젝트였던 것 같다.

그래도 어느 정도의 성과를 나열해보자면

✅ RAG 를 활용한 LLM 할루시네이션 축소<br>
✅ Langchain을 활용한 모델 서빙 방법<br>
✅ transformers pipeline 외 프롬프트 템플릿을 이용한 tools 정의법

정도 인 것 같다.

## 4. 후기

앞써 언급했 듯 강의 자료 + AI 활용으로 langchain + streamlit 으로 모델 서빙의 골조 정도를 알아본 프로젝트였다.

이후, `airllm` 을 사용하여 필자의 pc 에서 llm 모델을 구동하는 프로젝트를 진행하여 모델 서빙하는 방법에 대해 숙달해보는 시간을 가져 보아야겠다.
# Llama Index 코드설명(한슬)


## Overview

- 지금 작성된 코드는 Index, Pre-retrieval, Retrieval, Post-Retrieval, generation 모듈로 구성되어 있습니다
- (Architecture.md 파일에는 Query, Retrieval, Augment, Generate 이렇게 나눠주신것 같은데 여기에 맞춰 수정하겠습니다.
- Main-workflow에는 index를 제외한 4개의 모듈이 들어갑니다.
- 각 모듈도 workflow로 구성했습니다

<p align="center">
  <img src="[https://rfcjwfcfjbykdpmgkmwm.supabase.co/storage/v1/object/public/images/uploads/1776393676420-7wyfoy.webp](https://rfcjwfcfjbykdpmgkmwm.supabase.co/storage/v1/object/public/images/uploads/1776393403106-5n4xh7.webp)">
</p>

---

## Sub-workflow

- sub workflow(모듈)에는 각 기능을 담당하는 operator들이 들어갑니다. operator는 최소 기능을 수행하는 객체들입니다.
- architecture.md 파일에 query-plan, search-sourch 이런 기능들을 operator가 담당하게 될 것 같습니다.
- 그래서 각 sub-workflow에 operator들을 구현해서 넣고 빼고 하면서 테스트를 해 볼 예정이었습니다.

<p align="center">
  <img src="https://rfcjwfcfjbykdpmgkmwm.supabase.co/storage/v1/object/public/images/uploads/1776393676420-7wyfoy.webp">
</p>


## 코드 - main_workflow

```python
# modular_rag_main.ipynb
class MainWorkflow(Workflow):
    def __init__(self, 
                 pre_workflow: PreWorkflow,
                 retriever_workflow: RetrieverWorkflow,
                 post_workflow: PostWorkflow,
                 generate_workflow: GenerateWorkflow, 
                 *args, **kwargs):
        
        super().__init__(*args, **kwargs)
        # 외부에서 만든 커스텀 클래스 모듈들을 장착합니다.
        self.pre_workflow = pre_workflow
        self.retriever_workflow = retriever_workflow
        self.post_workflow = post_workflow
        self.generate_workflow = generate_workflow

    @step
    @dispatcher.span
    async def PreModule(self, ev: StartEvent) -> PreDoneEvent:
        query = ev.get("query")
        results = await self.pre_workflow.run(query=query)

        query = results['query']
        return PreDoneEvent(query=query)
```

- main-workflow에서는 각 모듈을 @step 데코레이터로 구분해놨고, 여기서 각 @step은 sub-workflow를 수행하는 기능만합니다.



## 코드 - sub_workflow

```python
# modular_rag_main.ipynb
dummy_operator = DummyOperator()
pre_operators = {"pre_step":[dummy_operator]}
pre_retriever_workflow = PreWorkflow(operators=pre_operators)
```
```python
# sub_workflows.py
class PreWorkflow(Workflow):
    def __init__(self,  operators:dict,**kwargs):
        super().__init__(**kwargs)
        self.operators = operators
        
    @step
    async def pre_step(self, ev: StartEvent) -> StopEvent:
        query = ev.get("query")
        operator_inputs ={'query': query} 
        operators = self.operators['pre_step'] # pre_step에서 필요한 operator만 가져옵니다.
        
        # operator들을 순차적 실행합니다.
        for operator in operators:
            operator_inputs = operator._operate(operator_inputs)
        
        # result = {'query' : str}
        return StopEvent(result=operator_inputs)
```


- sub-workflow에는 어느 @step에서 어떤 operator들을 사용할지 dictionary 형태로 입력으로 줍니다.
- sub-workflow는 이 operator list를 받아서 순서대로 실행하고 마지막에 결과를 반환합니다.

## 테스트
- index_module.inpynb 파일을 순서대로 실행하면 Qdrant 데이터베이스가 생성됩니다.
- 데이터베이스 생성이 끝나면 modular_rag_main.ipynb파일을 실행하면 전체 workflow가 실행됩니다.

  
## version

- llama-index : 0.14.15
- Qdrant-clinet : 1.16.2
- torch : 2.8.0+cu128
 










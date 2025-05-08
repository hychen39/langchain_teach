---
title: "RouterRunnable 物件"
export_on_save:
  html: true
---

# RouterRunnable 物件

## 使用情境

LLM 應用程式需要依據使用者不同的輸入, 判斷使用者的需求, 並且選擇適當的執行步驟來處理使用者的請求

例如:
- 執行英文翻譯成中文, 或者
- 執行中文翻譯成英文

## 工作原理

1. 開發者定義好一組 {key: Chain} 的字典
2. 使用 LLM 分析使用者的輸入, 決定合適的 key 值
3. 使用 key 值從字典中取得對應的 Chain
4. 執行 Chain, 並且回傳結果

## RouterRunnable 的使用範本

```
Input -> Runnable -> (RouterInput) -> RouterRunnable -> (Output) 
```

- 第一個 `Runnable` 要依據使用者的輸入, 決定要執行那一個工作, 取得該工作對應的 key 值. 
  - 此 `Runnable` 執行後必須輸出一個 `RouterInput` 物件。
  - `RouterInput` 物件必須包含一個 `key` 屬性, 這個屬性是字典中對應的 key 值及 `input` 屬性, 做為被挑選到的 `Runnable` 的輸入參數。

- `RouterRunnable` 物件會依據 `RouterInput` 物件的 `key` 屬性, 從字典中取得對應的 `Runnable` 物件.
  - 之後再從 `RouterInput` 物件中取得 `input` 屬性, 做為該 `Runnable` 的輸入參數, 並執行該 `Runnable` 以獲得結果。

## 分派工作的 `Runnable` 物件

- 工作分派的 `Runnable` 物件負責分析使用者的輸入, 並且決定要執行的工作。
- 輸出的資料型態為 `RouterInput` 物件, 供 `RouterRunnable` 物件使用。

### 設定能判別 "中翻英" 或 "英翻中" 的 `Runnable` 物件

PromptTemplate:

```python
task_dispatcher_prompt = """
You are a task dispatcher. You decide what task to do based on the input language.

You first check the what kind of language the input is, English or Chinese, or other.
Then,  You will return a JSON object **with no additional explanation** with the following fields:
- "key": the task key
- "input": the user's input

<input>
{user_input}

<Available tasks (task_key:rules)>
- `eng`: When the input is in Chinese, translate it to English.
- `chinese`: When the input is in English, translate it to Chinese.
- `default`: For other cases, it's not supported language.
"""
```

---

Prompt 中各段落的說明:
1. 第一段是系統提示, 提供給 LLM 了解它的角色是什麼
2. 第二段是 LLM 的工作說明, 說明步驟及輸出格式
3. 第三段是 LLM 的輸入, 這裡的 `<input>` 是一個占位符, 會在執行時被替換成使用者的輸入
4. 最後一段是可用的工作清單, 這裡的 `task_key` 是工作分派的 key 值, `rules` 是 LLM 判斷的規則。
   - `eng`: 當輸入是中文時, 翻譯成英文
   - `chinese`: 當輸入是英文時, 翻譯成中文
   - `default`: 其他情況, 不支援的語言

---

要求輸出為 JSON 格式的物件, 所以 Chain 中使用 `JSONOutputParser` 物件來解析輸出結果。

此 工作分派的 chain 物件如下:

```python
from langchain_core.output_parsers import JsonOutputParser, StrOutputParser
from langchain_core.prompts import PromptTemplate


dispatcher_runnable = (
    PromptTemplate.from_template(task_dispatcher_prompt)
    | llm
    | JsonOutputParser()
)
```

---

- 組合多個 `Runnable` 的 Chain 仍是一個 `Runnable` 物件, 可以再繼續組合成更複雜的 Chain。



## RouterRunnable 物件

假設已經建立兩個翻譯的 runnable 物件, 分別是 `ch_to_eng_runnable` 和 `eng_to_ch_runnable`
- 兩個 runnable 物件的輸入參數都是 `str` 型態, 輸出參數都是 `str` 型態

- 建立 `RouterRunnable` 物件時, 要提供一個字典
  - key: `task_key` 的值
  - value: `Runnable` 物件
- 這個字典的 key 值必須與工作分派的 `Runnable` 物件輸出的 `key` 值相同

--- 

Router 中的 runnable dictionary 物件如下:

```python
runnable_dict = {
    "eng": ch_to_eng_runnable,
    "chinese": eng_to_ch_runnable,
    "default": RunnableLambda(
        lambda x: "不支援的語言, 請使用中文或英文"
    )
}
```

- `RunnableLambda` 是將一般的 Python 函式轉換成 `Runnable` 物件。如此我們就可以 Chain 中使用一般的 Python 函式了

--- 

使用上述的 `runnable_dict` 物件, 建立 `RouterRunnable` 物件如下:

```python
from langchain_core.runnables import RouterRunnable
router_runnable = RouterRunnable(

    runnable_dict)
```


### 組裝完成的 Chain

依先前的 RouterRunnable 範本, 組裝出的 "英翻中" 或 "中翻英" 的 Chain 如下:

```python
translate_chain = (
    dispatcher_runnable
    | router_runnable
    | StrOutputParser()
)
```

# Demo Codes

See [RouterRunnable Example](demo_codes/router_runnable_example.ipynb) for the complete example.



---
title: "Cursor中使用DeepSeek的API"
description: "Cursor中使用DeepSeek的API"
category: ai
tags: Cursor,DeepSeek,AI
---

作为近期热点的DeepSeek 推理模型，如何使用到 Cursor 中助力编程，2 分钟带你快速配置。

## 前提

申请好DeepSeek API的key

## 添加Model Name

![截屏2025-01-26 14.35.21.png](/assets/images/截屏2025-01-26%2014.35.21.png)

* 在 Cursor Settings 中的 Models 中添加下面两个DeepSeek的模型名字

```swift
deepseek-chat    //V3 模型

deepseek-reasoner    //推理模型
```

来源：https://api-docs.deepseek.com/zh-cn/

> DeepSeek API 使用与 OpenAI 兼容的 API 格式，通过修改配置，您可以使用 OpenAI SDK 来访问 DeepSeek API，或使用与 OpenAI API 兼容的软件。
>
> | PARAM      | VALUE                                                        |
> | ---------- | ------------------------------------------------------------ |
> | base_url * | `https://api.deepseek.com`                                   |
> | api_key    | apply for an [API key](https://platform.deepseek.com/api_keys) |
>
> \* 出于与 OpenAI 兼容考虑，您也可以将 `base_url` 设置为 `https://api.deepseek.com/v1` 来使用，但注意，此处 `v1` 与模型版本无关。
>
> \* **`deepseek-chat` 模型已全面升级为 DeepSeek-V3，接口不变。** 通过指定 `model='deepseek-chat'` 即可调用 DeepSeek-V3。
>
> \* **`deepseek-reasoner` 是 DeepSeek 最新推出的[推理模型](https://api-docs.deepseek.com/zh-cn/guides/reasoning_model) DeepSeek-R1**。通过指定 `model='deepseek-reasoner'`，即可调用 DeepSeek-R1。



* 然后勾选✓这两项。

## 设置 OpenAI API Key

![截屏2025-01-26 14.40.28.png](/assets/images//截屏2025-01-26%2014.40.28.png)

* 在 Base Url 填入`https://api.deepseek.com`，然后点击 Save
* 在 OpenAI Key 填入 DeepSeek 的 key
* 点击 Verify 测试，通过就可以正常使用了


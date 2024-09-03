---
title: 阅读 MQTTX 项目：Desktop 【1】
date: 2024-09-03 22:37:10
categories: 记录
tags:
    - mqtt
    - TypeScript
    - avro
excerpt: 阅读 MQTTX(CLI) 中产生的理解和疑惑
cover: https://raw.githubusercontent.com/emqx/MQTTX/main/assets/mqttx-logo.png
---

## 特意声明的 'null'

-   注意到在全局类型生命中的 `ScriptModel` :

    ```TypeScript
    interface ScriptModel {
        id?: string
        name: string
        script: string
        type?: string | null
    }
    ```

-   末尾处 `type?: string | null` 对于 `type` 成员的类型声明，既然冒号前已经有了代表该成员可有可无的问号，为何还需要特意声明一个 `null`？
-   此外，在项目中还发现了许多已经带有问号标示的成员类型中特意声明了 `null` 的情况，甚为奇怪。

---

-   使用 telescope-fzf 插件寻找对于该类型的判断未果：

    ![没有找到对于该类型 null 值的判断](https://s2.loli.net/2024/09/03/1MHECf8ArXl7gSh.png)

## 摩西分海： `ScriptModel` 和 `ScriptState` 的拆分

-   注意到，当前的 MQTTX 类型声明中将所有的 `script` 作为同一个 interface:

    ```TypeScript
    // Scripts
    interface ScriptModel {
        id?: string
        name: string
        script: string
        type?: string | null
    }

    interface ScriptState {
        apply: MessageType
        function?: ScriptModel | null
        schema?: ScriptModel | null
        // config of function/schema
        config?: {
            // protobuf name
            name?: string
        }
    }
    ```

-   这种类型定义十分不利于后续 avro 编码类型的添加，因为其中许多的成员都可为 absent(undefined) 的状态。
-   另外，按照在 cli 部分中的开发经验来看，不光这二者应当被分开，Schema 部分也需要进一步细分为 Protobuf 和 Avro，由此可以发挥 TypeScript 的优势并完善类型检查。

    ![类型判断混淆](https://s2.loli.net/2024/09/03/sK6PzOtcgABkmrZ.png)

---

-   在修改 `src/views/script/index.vue` 组件的过程中，我已经将 `ScriptList` 类型拆分成了 `FunctionList` 与 `SchemaList` 两种类型，但是此处的 `ScriptModel` 与 `ScriptState` 在整个项目中的应用非常广泛（见下图），贸然修改可能会造成许多意想不到的 bug，所以我想应该在开始动手重构该部分之前询问一下老师。

    ![数量众多的 Reference](https://s2.loli.net/2024/09/03/Vk9KFcSCvWg1uQd.png)

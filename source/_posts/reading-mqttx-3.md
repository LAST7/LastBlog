---
title: 阅读 MQTTX 项目：Desktop 【1】
date: 2024-09-03 22:37:10
categories: 开发记录
tags:
    - mqttx
    - TypeScript
    - avro
excerpt: 阅读 MQTTX(Desktop) 中产生的理解和疑惑
cover: https://raw.githubusercontent.com/emqx/MQTTX/main/assets/mqttx-logo.png
---

## 特意声明的 'null'

-   注意到在全局类型声明中的 `ScriptModel` :

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

## 未定义的参数

-   在 `src/utils/protobuf.ts` 中可以看到这样一个函数：

    ```TypeScript
    export const deserializeBufferToProtobuf = (
        payload: Buffer,
        proto: string | undefined,
        protobufMessageName: string | undefined,
        to?: PayloadType,
    ): Buffer | string | undefined => {
        if (proto && protobufMessageName) {
            try {
                const root = protobuf.parse(proto).root
                const Message = root.lookupType(protobufMessageName)
                const MessageData = Message.decode(Buffer.from(payload))
                const err = Message.verify(MessageData)
                if (err) {
                    throw new SyntaxError(`Message deserialization error: ${err}`)
                }
                if (to) {
                    return Buffer.from(JSON.stringify(MessageData.toJSON()))
                }
                // return MessageData
                return protobufMessageName + ' ' + printObjectAsString(MessageData)
            } catch (error) {
                let err = transformPBJSError(error as Error)
                throw new SyntaxError(err.message.split('\n')[0])
            }
        }
    }
    ```

-   可以看到，其参数部分：

    ```TypeScript
    export const deserializeBufferToProtobuf = (
        payload: Buffer,
        // HERE:
        proto: string | undefined,
        // HERE:
        protobufMessageName: string | undefined,
        to?: PayloadType,
    // HERE:
    ): Buffer | string | undefined => {
    ```

-   对于 `proto`(raw schema) 和 `protobufMessageName` 的类型约束居然允许 `undefined` 出现，这让我感到有些不可思议：**这两个参数对于 Protobuf 的解码是必须的，既然都要调用这个解码函数了，不应该由上层代码（在参数输入时）来保证这两个参数的有效性么？**
-   此外，这两个参数如果其中出现了一个 `undefined`，代码居然没有设计任何报错的机制，任由函数平滑退出，返回另一个 `undefined`，颇为奇异。

-   该函数返回 `undefined` 后，调用其的上层代码还需要额外进行验证：

    ```TypeScript
    // ...
    const result = deserializeBufferToProtobuf(
        Buffer.from(Message.encode(Message.create(content)).finish()),
        proto,
        name,
    )
    // HERE:
    if (!result) {
        return ''
    }
    return result.toString()
    // ...
    ```

-   因此我认为，该参数的类型不应允许 `undefined` 出现。至于其验证，应由上层代码完成。

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
-   我个人倾向于将其全部拆分。


-   数量众多的 Reference:

    ![数量众多的 Reference](https://s2.loli.net/2024/09/03/Vk9KFcSCvWg1uQd.png)

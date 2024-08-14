---
title: 阅读 MQTTX 项目：CLI 【2】
date: 2024-08-14 17:02:26
categories: 记录
tags:
    - mqtt
    - avro
excerpt: 阅读 MQTTX(CLI) 中产生的理解和疑惑
cover: https://raw.githubusercontent.com/emqx/MQTTX/main/assets/mqttx-logo.png
---

## 信息的格式转换与 Avro/Protobuf 编码之间的冲突

-   以发送消息的流程为例，当前发送消息前的预处理中，格式转换的顺序是要先于 schema-based 编码的。

-   `pub.ts` :

    ```TypeScript
    const processPublishMessage = (
      message: string | Buffer,
      schemaOptions: SchemaOptions,
      format?: FormatType,
    ): Buffer | string => {
      const convertMessageFormat = (msg: string | Buffer): Buffer | string => {
        if (!format) {
          return msg
        }
        const bufferMsg = Buffer.isBuffer(msg) ? msg : Buffer.from(msg.toString())
        return convertPayload(bufferMsg, format, 'encode')
      }

      const serializeProtobufMessage = (msg: string | Buffer): Buffer | string => {
        if (schemaOptions.protobufPath && schemaOptions.protobufMessageName) {
          return serializeProtobufToBuffer(msg.toString(), schemaOptions.protobufPath, schemaOptions.protobufMessageName)
        }
        return msg
      }

      // ---HERE---
      const pipeline = [convertMessageFormat, serializeProtobufMessage]
      // ---HERE---

      return pipeline.reduce((msg, transformer) => transformer(msg), message) as Buffer
    }
    ```

-   可以看到 `pipeline` 的处理顺序中 `convertMessageFormat` 先于 `serializeProtobufMessage` 。

-   然而，基于 Schema 的编码方式所接受的 `message` 只能是 `javascript object`，不然程序就无法判断其内容是否符合 Schema Type，更不能将其中内容按照 Schema 编码成 Buffer 了。因此，此处 `protobuf` 的编码能够运作的前提是 `format` 为空（undefined）。

-   假设用户将消息的发送格式设置为 `hex` 或是 `cbor` 等编码格式，经过 `convertPayload` 处理后产生的 `Buffer` 将无法用于 avro 的编码。

-   **在我看来，这两者恐怕不应当被同时使用。**

---

-   以上纯属个人的一己之见，倘若有理解错误的地方，还希望老师予以指正。

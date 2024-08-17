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

-   然而，基于 Schema 的编码方式所接受的 `message` 只能是 `javascript object`，不然程序就无法读取并判断其内容是否符合 Schema Type，更不能将其中内容按照 Schema 编码成 Buffer 了。
-   因此，此处 `protobuf` 的编码能够运作的前提是 `format` 为空（undefined）或者能使用 `toString()` 方法进行还原的 Buffer。

-   假设用户将消息的发送格式设置为 `hex` 或是 `cbor` 等编码格式，经过 `convertPayload` 处理后产生的 `Buffer` 将无法用于 protobuf/avro 的编码。

-   可以看到，`serializeProtobufToBuffer` 函数中：

    ```TypeScript
    export const serializeProtobufToBuffer = (
        raw: string | Buffer,
        protobufPath: string,
        protobufMessageName: string,
    ): Buffer => {
        let rawData = raw.toString('utf-8') // ???
        let bufferMessage = Buffer.from(rawData) // ???
        try {
            const root = protobuf.loadSync(protobufPath)
            const Message = root.lookupType(protobufMessageName)
            const err = Message.verify(JSON.parse(rawData)) // ???
        if (err) {
            logWrapper.fail(`Message serialization error: ${err}`)
            process.exit(1)
        }
            const data = Message.create(JSON.parse(rawData)) // ???
            ...
        }
        ...
    }
    ```

    -   该函数的参数 `raw` 被设为了 `string | Buffer` 类型。很奇怪的是，此后的几行代码似乎都在试图将其从 `Buffer` 还原为 `string`，再使用 `JSON.parse(rawData)` 将其还原为一个 JavaScript Object。
    -   为什么要这样？直接设定传入的 `raw` 为 `string` 或者更干脆的设为 `object` 好像更加简单直接。
    -   _当然，这里面可能有一些以我的水平没有考量到的情况，请予以指正。_

---

-   **在我看来，格式转换和基于 Schema 的编码恐怕不应当被同时使用。**

## _string_ 和 _Buffer_ 类型的混用

-   注意到，在发送的消息中，对于 `message` 的类型是这样定义的：

    ```TypeScript
    interface PublishOptions extends ConnectOptions {
        topic: string
        // ---HERE---
        message: string | Buffer
        // ---HERE---
        qos: QoS
        retain?: boolean
        ...
    }
    ```

-   对于这个 `message` 变量的类型，为什么他不能只是 `string`？**到底在什么情况下他会以一个 `Buffer` 类型从命令行输入？**
-   该 _Union Type_ 在后续的消息处理中带来了一些麻烦，我想了解他在何种情况下会作为 `Buffer` 类型输入，以及如果可能的话，是否能直接将其类型直接修改为 `string`。

---

**更新：**

-   了解到在使用了 `--file-read` option 后，消息主体会从文件中读取。读取文件的方法是 `fs.readFileSync`，该方法在未设置第二项参数时返回值的类型为 `Buffer`。
-   请问能否将其更改为传回 `string` 的形式？当前 `processPublishMessage` 方法中 `string` 和 `Buffer` 的混用实在是令我感到些许反感。

-   虽然使用 `string` 作为返回值的 `fs.readFileSync` 在性能上大概降低了 75% 左右，但是我认为在 `serializeProtobufToBuffer` 等众多函数中因为不能确定 `msg` 的类型而进行反反复复的 `JSON.parse()` 和 `toString()` 操作更加消耗性能（例如上个段落中 `serializeProtobufToBuffer` 里面那样，以及诸多的格式转换函数中频繁的 `toString()`）。

-   相关测试请看：

{% folding blue::性能测试 %}

下面是一个使用 ChatGPT 生成的简易测试程序，用于测试 `fs.readFileSync` 返回 `Buffer` 类型和 `string` 类型的性能差异：

```TypeScript
const fs = require("fs");
const path = require("path");
const { performance } = require("perf_hooks");

// Function to test reading file as Buffer
function readAsBuffer(filePath) {
  const start = performance.now();
  for (let i = 0; i < 10000; i++) {
    const data = fs.readFileSync(filePath); // Returns a Buffer
  }
  const end = performance.now();
  return end - start;
}

// Function to test reading file as String
function readAsString(filePath) {
  const start = performance.now();
  for (let i = 0; i < 10000; i++) {
    const data = fs.readFileSync(filePath, "utf8"); // Returns a String
  }
  const end = performance.now();
  return end - start;
}

// File path for testing
const filePath = path.join(__dirname, "testfile.txt");

// Write a large test file
const largeContent = "This is a test file.\n".repeat(10000);
fs.writeFileSync(filePath, largeContent);

// Test performance for Buffer
const bufferTime = readAsBuffer(filePath);
console.log(`Buffer read time: ${bufferTime.toFixed(2)} ms`);

// Test performance for String
const stringTime = readAsString(filePath);
console.log(`String read time: ${stringTime.toFixed(2)} ms`);

// Clean up the test file
fs.unlinkSync(filePath);
```

测试结果：

```plaintext
$ node fsType.js
Buffer read time: 389.62 ms
String read time: 1379.63 ms
```

可以看到，性能差距的确相当巨大。

---

但是，只要在以 `Buffer` 类型读取的数据后添加一行 `toString()` 方法：

```TypeScript
...
function readAsBuffer(filePath) {
  const start = performance.now();
  for (let i = 0; i < 10000; i++) {
    const data = fs.readFileSync(filePath); // Returns a Buffer
    // Add this line
    const stringData = data.toString();
  }
  const end = performance.now();
  return end - start;
}
...
```

他们的性能就会变得相差无几：

```plaintext
$ node fsType.js
Buffer read time: 1357.50 ms
String read time: 1369.35 ms
```

---

所以，我建议直接使用 `string` 作为返回值。

{% endfolding %}

---

-   或者我们还可以选择另一个方案：为 `Buffer` 类型的消息发送建立一个新的消息发送数据流，在分离 `msg` 为 `string` 和 `Buffer` 这两种情况的同时也保持良好的文件读写性能。
-   当然，这会需要更多的开发投入。

---

-   以上纯属个人的一己之见，倘若有理解错误的地方，还希望老师予以指正。

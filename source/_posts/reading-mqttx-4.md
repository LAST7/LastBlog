---
title: 阅读 MQTTX 项目：Protobuf Test Case
date: 2024-09-05 19:34:00
categories: 开发记录
tags:
    - mqttx
    - TypeScript
    - protobuf
    - jest
excerpt: 关于 MQTTX 中对于 protobuf utils 的测试用例的一些观察和意见
cover: https://raw.githubusercontent.com/emqx/MQTTX/main/assets/mqttx-logo.png
---

## 前言

-   本人在编写针对 `src/utils/protobuf.ts` 的过程中发现该测试文件已经完成，因此将其 Sync 后 Pull 到本地与本人写的测试代码进行比对，发现了一些不妥当的地方。
-   **当然，也可能是本人意见有误，望海涵。**

## 另：工作的必要性考虑

-   由于本人参与的开源之夏项目只包含 MQTTX 项目中 avro 编码的功能开发，protobuf 功能实际并非由该项目负责，也没有义务指导我开发这部分的功能。
-   **因此，倘若认为 protobuf 部分的测试用例无须修改，或者生活工作过于忙碌分身乏术，可以直接忽略本文章以及本次本人提交的 PR。**

## 测试思路

-   以其中一个测试用例为例：

    ```TypeScript
    it('should serialize JSON message with a protobuf schema correctly', () => {
        const resultBuffer = serializeProtobufToBuffer(jsonMessage, mockProtoPath, 'SensorData')
        expect(resultBuffer).toBeInstanceOf(Buffer)

        const root = protobuf.parse(protoSchema).root
        const SensorData = root.lookupType('SensorData')
        const decodedMessage = SensorData.decode(resultBuffer)
        expect(decodedMessage.toJSON()).toEqual({
            deviceId: '123456',
            sensorType: 'Temperature',
            value: 22.5,
            timestamp: '16700',
        })
    })
    ```

-   这里的思路很奇怪，详见下图：

    ![test_routine](https://s2.loli.net/2024/09/05/CWQbiGVxa8u92Fe.png)

-   私以为，我们应当将**相同的输入**分别输入**待测试的流程**和**保证正确的流程**，并将这两个流程的输出进行比较，相同则为通过，不同则为不通过。
-   但是上面的这个测试流程如图所示，将**待测试的编码流程**产出的结果输入了**解码流程**，而后将其结果与一个硬编码对象进行比较，在我看来有些奇怪。
-   后续还有许多相似的点，不再赘述。

## _toHaveBeenCalledWith_ 的陷阱

### 源头

-   注意到 `src/utils/protobuf.ts` 中的 42-43 行并未被测试覆盖到：

    ```
    ---------------------|---------|----------|---------|---------|----------------------
    File                 | % Stmts | % Branch | % Funcs | % Lines | Uncovered Line #s
    ---------------------|---------|----------|---------|---------|----------------------
    All files            |    23.5 |    10.33 |   19.12 |   23.85 |
        ...
      protobuf.ts        |   94.44 |    66.66 |     100 |   94.11 | 42-43
        ...
    ```

-   翻阅源码，42-43 及其周围代码如下：

    ```TypeScript
    // ...
    const root = protobuf.loadSync(protobufPath)
    const Message = root.lookupType(protobufMessageName)
    const MessageData = Message.decode(payload)
    const err = Message.verify(MessageData)
    if (err) {
      // line 42:
      logWrapper.fail(`Unable to deserialize protobuf encoded buffer: ${err}`)
      // line 43:
      process.exit(1)
    }
    // ...
    ```

-   显然，未被触发的 branch 与 `Message.verify(...)` 函数有关。
-   但是，在 `serializeProtobufToBuffer` 函数中也有相同的分支，为什么它被认为受到测试覆盖了？

---

### 原因

-   为篇幅考虑，只说结论：

    1.  `toHaveBeenCalledWith(...)` 确实用于判断某指定的 mock function 是否被调用过，**但是他的判断范围是全局的！会包括该函数在其他的测试用例中被调用的次数**！
        {% folding blue::案例 %}

        ```TypeScript
        // BUG: this would pass
        it('nothing happens', () => {
            expect(mockExit).toHaveBeenCalledWith(1)
        })

        it('should log an error and exit if message does not match schema', () => {
            const invalidMessage = Buffer.from('{"invalidField": "value"}')
            serializeProtobufToBuffer(invalidMessage, mockProtoPath, 'SensorData')
            expect(mockExit).toHaveBeenCalledWith(1)
        })
        ```

        -   第一个测试用例将会通过，因为对于 `mockExit` 的追踪是全局的，第二个测试用例中触发的次数在第一个测试用例中也会被计算。

        {% endfolding %}

    2.  在 `serializeProtobufToBuffer` 函数中，两个分支内 `logWrapper.fail()` 中的 error message 以相同的形式开头：`Message serialization error: ...`。**配合前一个因素**，导致了一个 BUG：**只要其中一个 `logWrapper.fail()` 被调用，Jest 会认为两个测试用例都已通过**。
        -   这种情况当然也出现在 `process.exit(1)` 上，但是影响相对较小。
    3.  `protobuf.Type.verify` 的行为并非预期（见下章节）。

---

### 解决方案

1.  对于不同的错误处理分支，应当编写不同的 error message。
2.  在每一个测试前重置 mock function 的状态（代码如下）

    ```TypeScript
    describe("...", () => {
        beforeEach(() => {
            jest.clearAllMocks()
        })
    })
    ```

    {% notel orange fa-clock 更新 %}
    观察到在其他测试中都调用了该函数，但是不知为何在 `protobuf.test.ts` 中没用调用。
    猜测是 copy-paste 导致的（bushi
    {% endnotel %}

3.  使用 `toHaveBeenNthCalledWith` _（不推荐，该函数的行为很奇怪）_

-   **个人认为，第一和第二种方案都应当被实施。**

---

-   但是，这并没有回答我们最开始的问题：

> 显然，未被触发的 branch 与 `Message.verify(...)` 函数有关。
> 但是，在 `serializeProtobufToBuffer` 函数中也有相同的分支，为什么它被认为受到测试覆盖了？

-   不必着急，下一篇章将会讨论这个问题。

## _protobuf.Type.verify_ 的意外行为

### 错误表现

-   在修复了上述问题后，会发现许多原本通过的测试用例反而不通过了。他们无一例外都与 `protobuf.Type.verify` 有关。如下所示：

    ```TypeScript
    it('should log an error and exit if message does not match schema', () => {
      const invalidMessage = Buffer.from('{"invalidField": "value"}')

      // BUG: Can not trigger line 17-18 of protobuf.ts
      // `protobuf.Type.verify` seems to be working in an unexpected way.
      // With `invalidMessage` as `raw`, no error is triggerred.
      expect(() => {
        serializeProtobufToBuffer(invalidMessage, mockProtoPath, 'SensorData')
      }).toThrow()
      expect(logWrapper.fail).toHaveBeenCalledWith(
        expect.stringMatching(/Unable to serialize message to protobuf buffer:*/),
      )
      expect(mockExit).toHaveBeenCalledWith(1)
    })
    ```

    ```TypeScript
    it('should log an error and exit if buffer is not valid protobuf', () => {
        const invalidBuffer = Buffer.from([0x08, 0x96, 0x01]) // An invalid protobuf buffer

        // BUG: Can not trigger line 42-43 of protobuf.ts
        // `protobuf.Type.verify` seems to be working in an unexpected way.
        // With `invalidBuffer` as `payload`, the decoded message is '{"deviceId":""}' and no error is triggerred.
        expect(() => deserializeBufferToProtobuf(invalidBuffer, mockProtoPath, 'SensorData', true)).toThrow()
        // error message is generated by function `transformPBJSError`
        expect(logWrapper.fail).toHaveBeenCalledWith(expect.stringMatching(/Message deserialization error:*/))
        expect(mockExit).toHaveBeenCalledWith(1)
    })
    ```

    -   还有其他的测试用例无法通过，不再赘述。

### 原因

-   但是，有一个测试用例触发了 17-18 行的分支，并且成功通过：

    ```TypeScript
    // INFO: only this test case can trigger line 17-18
    it('should handle verification errors from protobuf', () => {
        const invalidMessage = '{"deviceId": 123}' // deviceId should be a string
        expect(() => serializeProtobufToBuffer(invalidMessage, mockProtoPath, 'SensorData')).toThrow()
        expect(logWrapper.fail).toHaveBeenCalledWith(expect.stringMatching(/Message serialization error:*/))
        expect(mockExit).toHaveBeenCalledWith(1)
    })
    ```

-   这让我产生了些许灵感，会不会是因为 `protobuf.Type.verify` 方法只能验证 Schema 中出现过的数据的**类型**是否正确？当出现一个完全不符合 Schema 定义的 protobuf 消息时，它会怎么处理呢？

-   测试如下：

    {% folding blue::测试 %}

    注：该测试由 ChatGPT 提供，较为冗长，建议直接快进到输出和结论部分。

    ```TypeScript
    import * as protobuf from "protobufjs";

    const protoSchema = `
    syntax = "proto3";

    message SensorData {
      string deviceId = 1; // deviceId is a required string field
      float temperature = 2; // temperature is a required float field
    }
    `;
    const root = protobuf.parse(protoSchema).root;
    const SensorData = root.lookupType("SensorData");

    // Utility function to test `verify` behavior
    function testVerify(data: any) {
      const err = SensorData.verify(data);
      if (err) {
        console.error(`Verification failed: ${err}`);
      } else {
        console.log("Verification succeeded:", data);
      }
    }

    // 1. Test with Correct Data
    console.log("\nTest 1: Correct Data");
    const correctData = { deviceId: "sensor-001", temperature: 22.5 }; // Modify based on your schema
    testVerify(correctData);

    // 2. Test with Incorrect Data Type
    console.log("\nTest 2: Incorrect Data Type");
    const incorrectTypeData = { deviceId: 123, temperature: 22.5 }; // deviceId should be a string
    testVerify(incorrectTypeData);

    // 3. Test with Missing Required Field
    console.log("\nTest 3: Missing Required Field");
    const missingFieldData = { temperature: 22.5 }; // Missing deviceId
    testVerify(missingFieldData);

    // 4. Test with Extra Field
    console.log("\nTest 4: Extra Field");
    const extraFieldData = {
      deviceId: "sensor-001",
      temperature: 22.5,
      humidity: 40,
    }; // humidity is not defined in schema
    testVerify(extraFieldData);

    // 5. Test with Completely Invalid Data
    console.log("\nTest 5: Completely Invalid Data");
    const completelyInvalidData = { invalidField: "value" }; // No matching fields in schema
    testVerify(completelyInvalidData);

    // 6. Test with Completely Invalid Data
    console.log("\nTest 6: Unstructured message");
    const notJSONMessage = "Hello, world!";
    testVerify(notJSONMessage);

    // 7. Test with Random Buffer (Invalid Protobuf Message)
    console.log("\nTest 7: Random Buffer");
    const randomBuffer = Buffer.from([0x08, 0x96, 0x01]);
    try {
      const decodedMessage = SensorData.decode(randomBuffer);
      console.log("Decoded message:", decodedMessage);
      testVerify(decodedMessage);
    } catch (error) {
      console.error("Decoding failed:", error);
    }

    // 8. Test with Stringified JSON (Invalid Protobuf Message)
    console.log("\nTest 8: Stringified JSON");
    const stringifiedJson = '{"deviceId": "sensor-001"}';
    try {
      const decodedMessage = SensorData.decode(Buffer.from(stringifiedJson));
      console.log("Decoded message:", decodedMessage);
      testVerify(decodedMessage);
    } catch (error) {
      console.error("Decoding failed:", error);
    }

    ```

    输出：

    ```plaintext
     yarn start
    yarn run v1.22.22
    $ node build/index.js

    Test 1: Correct Data
    Verification succeeded: { deviceId: 'sensor-001', temperature: 22.5 }

    Test 2: Incorrect Data Type
    Verification failed: deviceId: string expected

    Test 3: Missing Required Field
    Verification succeeded: { temperature: 22.5 }

    Test 4: Extra Field
    Verification succeeded: { deviceId: 'sensor-001', temperature: 22.5, humidity: 40 }

    Test 5: Completely Invalid Data
    Verification succeeded: { invalidField: 'value' }

    Test 6: Unstructured message
    Verification failed: object expected

    Test 7: Random Buffer
    Decoded message: SensorData { deviceId: '' }
    Verification succeeded: SensorData { deviceId: '' }

    Test 8: Stringified JSON
    Decoding failed: RangeError: index out of range: 3 + 100 > 26
        at indexOutOfRange (/home/last/Coding/mqtt/testProto/node_modules/protobufjs/src/reader.js:13:12)
        at BufferReader.skip (/home/last/Coding/mqtt/testProto/node_modules/protobufjs/src/reader.js:343:19)
        at Reader.skipType (/home/last/Coding/mqtt/testProto/node_modules/protobufjs/src/reader.js:369:18)
        at Reader.skipType (/home/last/Coding/mqtt/testProto/node_modules/protobufjs/src/reader.js:373:22)
        at Type.SensorData$decode [as decode] (eval at Codegen (/home/last/Coding/mqtt/testProto/node_modules/@protobufjs/codegen/index.js:50:33), <anonymous>:19:5)
        at Object.<anonymous> (/home/last/Coding/mqtt/testProto/build/index.js:90:39)
        at Module._compile (node:internal/modules/cjs/loader:1364:14)
        at Module._extensions..js (node:internal/modules/cjs/loader:1422:10)
        at Module.load (node:internal/modules/cjs/loader:1203:32)
        at Module._load (node:internal/modules/cjs/loader:1019:12)
    Done in 0.08s.
    ```

    {% endfolding %}

-   注意到：

    1.  对于 Schema 中存在的成员，倘若变量类型错误，`protobuf.Type.verify` 会正常报错。
    2.  倘若 Message 中缺少 Schema 中存在的成员，不会报错。
    3.  倘若 Message 中存在 Schema 中不存在的成员，不会报错。
    4.  **倘若 Message 完全不符合 Schema 定义，不会报错。**
    5.  **对于完全不符合 protobuf 编码格式的随机 Buffer，只要不超出限制的长度就不会报错。**

-   综上所述，`protobuf.Type.verify` 对于消息的检查并不严格，会遗漏很多情况。很多在测试中预期会产生报错的行为，实际不会产生报错，导致测试用例失败。

---

-   以上便是我对于 `protobuf.Type.verify` 函数行为的一些测试和总结，希望我已经将问题表述清楚了。

### 解决方案

-   说老实话，除去直接删除相关的测试用例，将实际情况交由用户处理之外，我想不出什么方法了。

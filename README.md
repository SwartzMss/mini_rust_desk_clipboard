# clipboard

通过网络复制文件和文本

## 工作原理

术语:

- cliprdr: 此模块
- local: 发起文件复制事件的一端
- remote: 粘贴来自 `local` 复制的文件的一端 

文件复制和粘贴的主要算法基于
[Remote Desktop Protocol: Clipboard Virtual Channel Extension](https://winprotocoldoc.blob.core.windows.net/productionwindowsarchives/MS-RDPECLIP/%5bMS-RDPECLIP%5d.pdf),
可以总结如下:

0. local 和 remote 互相通知已准备就绪。
1. local 订阅/监听系统剪贴板的文件复制事件。
2. local 收到文件复制事件后，通知 remote。
3. remote 确认收到并尝试拉取文件列表。
4. local 更新其文件列表，remote 将接收到的文件列表刷新到剪贴板。
5. remote 的操作系统或桌面管理器发起粘贴，使其他程序读取剪贴板文件。将这些读取请求转换为 RPC 调用：
   - 在 Windows 上，所有文件读取都会通过流文件 API 进行。

6. 一个接一个地完成所有文件的粘贴.

从网络数据传输的角度来看:

```mermaid
sequenceDiagram
    participant l as Local
    participant r as Remote
    note over l, r: 初始化
    l ->> r: Monitor Ready
    r ->> l: Monitor Ready
    loop 获取剪贴板更新
        l ->> r: Format List (我有更新)
        r ->> l: Format List Response (通知完成)
        r ->> l: Format Data Request (请求文件列表)
        activate l
            note left of l: 从系统剪贴板获取文件列表
            l ->> r: Format Data Response (包含文件列表)
        deactivate l
        note over r: 使用接收到的文件列表更新系统剪贴板
    end
    loop 某应用请求已复制的文件
        note right of r: 应用读取文件从 x 到 x+y
        note over r: 该文件是列表中的第 a 个文件
        r ->> l: File Contents Request (读取文件 a 偏移量 x 大小 y)
        activate l
            note left of l: 找到列表中的第 a 个文件，读取从 x 到 x+y
            l ->> r: File Contents Response (文件 a 偏移量 x 大小 y 的内容)
        deactivate l
    end
```

注意：在实际实现中，双方都可以发送剪贴板更新并请求文件内容。并没有只允许 local 更新剪贴板并将文件复制到 remote 的限制。

## 实现

### windows

![scene](./docs/assets/scene3.png)

![A1->B1](./docs/assets/win_A_B.png)

![B1->A1](./docs/assets/win_B_A.png)

该协议最初设计为 Windows RDP 的扩展，因此消息包的格式非常适合 Windows。

启动 cliprdr 时，会创建一个线程以创建一个不可见窗口并订阅 OLE 剪贴板事件。窗口的回调（见 src/windows/wf_cliprdr.c 中的 cliprdr_proc）被设置为处理各种事件。

详细实现见上图.

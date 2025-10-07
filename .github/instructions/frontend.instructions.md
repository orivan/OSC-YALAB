# 指导 GitHub Copilot - 前端开发规范 (OSC-YALAB)

当前端开发涉及到为 "雅典娜之盾" 项目构建 3D 可视化界面时，你必须严格遵循以下规范。

## 1. 核心技术栈

- **UI 框架**: **React (>= 18.0)**
- **3D 渲染库**: **Three.js**
- **集成方案**: **React Three Fiber (R3F)** (`@react-three/fiber`)
- **辅助工具集**: **Drei** (`@react-three/drei`)
- **状态管理**: **Zustand**
- **样式**: **Tailwind CSS**

## 2. 核心架构与开发原则【最高优先级】

### A. 声明式优先原则
**必须**使用 React Three Fiber (R3F) 提供的声明式 API 和组件来构建 3D 场景。
- **禁止**在 React 组件中直接编写命令式的 Three.js 代码 (例如 `scene.add(mesh)`)。
- **应当**使用 R3F 的 JSX 语法 (例如 `<mesh>`, `<boxGeometry>`) 来描述场景。
- **必须**使用 R3F 提供的钩子 (Hooks) 如 `useFrame`, `useThree`, `useLoader` 来处理动画循环、访问核心对象和异步加载资源。

### B. 彻底的组件化原则
将复杂的 3D 场景和 UI 拆分为小型的、可复用的、单一职责的 React 组件。
- **示例**: 一个汽车模型应被拆分为 `<Wheel>`, `<Chassis>`, `<Lights>` 等子组件。UI 控件和 3D 对象都应被视为组件。

### C. 状态管理原则
- **局部状态**: 使用 React 内置的 `useState` 管理组件内部的瞬时状态。
- **全局状态**: 使用 **Zustand** 创建 store 来管理跨组件共享的全局状态（例如场景配置、用户数据、后端返回数据等）。禁止通过属性逐层传递（props drilling）来共享全局状态。

### D. 与后端服务交互
- **必须**通过调用 `OSC-YALAB` 后端服务暴露的 **HTTP/JSON 接口**（由 Kratos gRPC-Gateway 提供）来获取数据。
- 使用 `fetch` API 或 `axios` 库发起网络请求，并妥善处理加载（loading）和错误（error）状态。

# 行为规则（Instinct）示例

> 原料来源：**工具调用观测**（工具名 + 参数 + 是否成功 + 轮内顺序）。
> 有价值规则的共同点：是一个可复用的**习惯/偏好/工作流**，而不是系统本就强制的动作（如"先读后改"）。
> 下面每条都先给"观察到的工具序列"，再给提炼出的规则。

---

## 1. 提交前先跑测试（testing）
观察序列反复出现：
```
EditFile(foo.java) -> Bash(mvn test) -> Bash(git commit)
WriteFile(bar.test.ts) -> Bash(npm test) -> Bash(git add .)
```
提炼规则：
```
/** intuition: testing */
- **Trigger:** 即将 commit
  **Action:** 先跑测试套件，失败则修复后再提交
  **Confidence:** 0.72
```
为什么有用：不是强制项，是你们（或团队）的实际工作流，跨会话该保持。

---

## 2. 格式化后提交（workflow）
观察序列：
```
WriteFile(a.ts) -> Bash(npx prettier --write a.ts)
EditFile(b.ts)  -> Bash(npx eslint --fix b.ts)
```
提炼规则：
```
/** intuition: workflow */
- **Trigger:** 写/改代码文件之后
  **Action:** 用格式化/检查工具统一处理后再继续
  **Confidence:** 0.68
```

---

## 3. 提交前自查 diff（git）
观察序列：
```
EditFile(x.java) -> Bash(git diff) -> Bash(git add .) -> Bash(git commit)
```
提炼规则：
```
/** intuition: git */
- **Trigger:** 有改动将要提交
  **Action:** 先 git diff 自查 diff，再 git add / commit
  **Confidence:** 0.70
```

---

## 4. 精确定位用 Grep 而非碰运气（filesystem）
观察序列：
```
Grep("findUserById") -> ReadFile(userRepo.java) -> EditFile(...)
```
提炼规则：
```
/** intuition: filesystem */
- **Trigger:** 需要定位某个符号/定义/引用
  **Action:** 用 Grep(全限定名) 配合 ReadFile 精确定位，而不是 Glob 盲目匹配
  **Confidence:** 0.66
```

---

## 5. 先搜后读再改（workflow，非强制的细粒度写法）
观察序列：
```
Grep(pattern) -> ReadFile(match) -> ReadFile(context) -> EditFile
```
提炼规则：
```
/** intuition: workflow */
- **Trigger:** 修改一处逻辑
  **Action:** 先搜索引用、再读取上下文文件，确认影响面后才动手编辑
  **Confidence:** 0.63
```
说明：这是把"读文件"的**习惯写得更有针对性**，而不是系统强制的"读过才能改"——多了一步"搜引用、读上下文"。

---

## 6. 自测工具在改动后立刻执行（testing）
观察序列：
```
EditFile(util.ts) -> Bash(node -e "import('./util')") -> Bash(npm test)
```
提炼规则：
```
/** intuition: testing */
- **Trigger:** 修改核心工具/公共函数
  **Action:** 顺手做一次冒烟自测，再跑完整测试
  **Confidence:** 0.64
```

---

## 7. 长命令拆成多步并行/串行执行（workflow）
观察序列（稳定地不在一条里 `&&` 链一堆）：
```
Bash(step 1) -> Bash(step 2) -> Bash(step 3)
```
提炼规则：
```
/** intuition: workflow */
- **Trigger:** 需要执行多条有先后依赖的命令
  **Action:** 拆成多个 Bash 调用逐步执行，而不是拼长链 && （便于定位失败）
  **Confidence:** 0.62
```

---

## 8. 读文件前先确认路径存在（filesystem，防御习惯）
观察序列：
```
Glob(pattern) -> ReadFile(glob 结果) -> WriteFile(...)
Bash(test -f x) -> ReadFile(x)
```
提炼规则：
```
/** intuition: filesystem */
- **Trigger:** 要读取/覆盖某个文件
  **Action:** 先用 Glob/存在性检查确认它存在，再读或写
  **Confidence:** 0.60
```

---

## 9. 修改接口时顺带搜调用方（feedback 修正型，从"做了之后才补"变成"一开始就做"）
观察序列（纠正后变得稳定）：
```
Grep("foo(") -> Grep("foo") -> ReadFile(caller1) -> ReadFile(caller2) -> EditFile(foo)
```
提炼规则：
```
/** intuition: workflow */
- **Trigger:** 修改了一个被多处调用的函数/API 签名
  **Action:** 先全仓搜全部调用点，确认不破坏其他模块后再改
  **Confidence:** 0.74
```

---

## 10. 构建/编译失败先看错误定位再改（testing）
观察序列：
```
Bash(mvn compile) -> Grep("ERROR") -> ReadFile(报错文件) -> EditFile(...)
```
提炼规则：
```
/** intuition: testing */
- **Trigger:** 编译/构建报错
  **Action:** 先精确 grep 错误行、读对应文件定位，再修改；不要盲目乱改
  **Confidence:** 0.67
```

---

## 总结：哪些能用这套模块总结出来
- **能**：工具选用偏好（Grep vs Glob）、工具序列工作流（查→读→改、测→格式化→提交）、防御性调用顺序（先确认再读写）、针对多次出现的动作的流程抽象。
- **不能**：纯代码约定、仓库特有名词、用户口头偏好——因为观测里没有这些信息（那些要靠 `MemoryManager`/`/rule` 去补）。
- 判断是否有用：**该序列是不是"可做可不做、但做了更好"的习惯**。若是系统/协议强制项（如必读后改），就是噪声规则。

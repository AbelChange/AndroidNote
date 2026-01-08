# 📚 English Vocabulary & Android Interview Expression Summary

本档整合了关键英语短语解释及 Android 技术面试相关口语表达，方便记忆与应用。

---

## 1. 关键英语短语详细解释

| 短语/词汇                                             | 解释                                       | 示例句                                                       |
| ----------------------------------------------------- | ------------------------------------------ | ------------------------------------------------------------ |
| **have a good grasp of sentences**                    | 掌握句子结构和语法，有较好理解和表达能力   | She has a good grasp of English grammar.                     |
| **convey  ideas** clearly                             | 清晰地表达想法                             | His sentences convey clear ideas about the project.          |
| **phrasing issues**                                   | 用词或表达方式存在问题，可能不够自然或地道 | There are some phrasing issues in your essay that need fixing. |
| **nothing that obscures meaning or causes confusion** | 语言问题不会导致意思不清或理解困难         | There are errors, but nothing that obscures meaning or causes confusion. |
| **polish native-like idiomatic phrases**              | 润色英语，使之更地道，学习并使用习惯用语   | I want to polish my English with native-like idiomatic phrases. |

---

## 2. Android 生命周期相关表达

| 英文表达                        | 中文含义     | 示例句                                                       |
| ------------------------------- | ------------ | ------------------------------------------------------------ |
| `lifecycle methods`             | 生命周期方法 | The lifecycle methods include `onCreate`, `onStart`, etc.    |
| `foreground / background`       | 前台 / 后台  | `onResume` puts the app in the foreground.                   |
| `resource-intensive operations` | 耗资源的操作 | You can pause resource-intensive operations in `onPause`.    |
| `destroyed by the system`       | 被系统销毁   | The activity might be destroyed by the system under low memory. |
| `save the current state`        | 保存当前状态 | Use `onStop` to save the current state.                      |
| `restored later`                | 后续恢复     | The UI can be restored later by the system.                  |

---

## 3. ViewModel 相关表达

---

`data is preserved`  数据被保留  Data in ViewModel is preserved even after recreation.



## 4. 语法与表达纠正建议

| 推荐表达          | 说明               | 示例                                               |
| ----------------- | ------------------ | -------------------------------------------------- |
| `as we know`      | 正确表达动词时态   | As we know, the view hierarchy might be destroyed. |
| `hierarchy`       | 拼写正确           | The view hierarchy may be recreated.               |
| `can be restored` | 被动语态用过去分词 | The UI can be restored later.                      |

---

## 5. 表达强化建议（更多句式）

- `A ViewModel is used to store and manage UI-related data in a lifecycle-aware way.`  
- `The view hierarchy can be destroyed during configuration changes, such as screen rotations, or when the system needs to free up memory.`  
- `Instead of putting data in Activities or Fragments, we use ViewModel to preserve it.`

---

## 6. 口语交流常用表达（补充）

| 短语                      | 中文含义   | 示例                                                         |
| ------------------------- | ---------- | ------------------------------------------------------------ |
| `launch process`          | 启动流程   | The app starts by initializing 3D rendering and fetching cloud data. |
| `parallel initialization` | 并行初始化 | 3D and cloud data modules initialize in parallel.            |
| `data rendering`          | 数据渲染   | After data is ready, it is sent to the 3D module for rendering. |
| `error handling`          | 异常处理   | Glide provides error handling when loading images fails.     |

---

## 7. 额外建议

- 通过“组”（row）和“列”（column）理解布局，减少对坐标轴映射的认知负担。  
- 保持用词简洁明了，避免多层抽象，便于快速表达和面试答题。

---

*如果你想，我可以帮你整理更多面试问题和答案，或者出练习题，欢迎随时告诉我！*  
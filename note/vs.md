# Cline VS Claude Code

总体上是大同小异。



## agentic loop 类似



## 细节上不同


|               | cline                       | claude code            |
| ------------- | --------------------------- | ---------------------- |
| checkpoint    | 使用 git 构建的 shadow repo | 自定义的 snapshot 管理 |
| sandbox       | 不支持                      | 不支持 windows         |
| rag           |                             |                        |
| 代码质量      | 规范、易于协作              | 高质量、精于优化       |
| 多 agent 架构 |                             |                        |

### claude code
tool call 的时候有并行执行。

遥测，引入风险。

windows 放弃治疗，不用沙箱
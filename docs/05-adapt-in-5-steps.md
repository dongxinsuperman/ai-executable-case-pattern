# 5 步适配到公司内部

公开版 skill 是 baseline，不建议原样用于公司业务。最短落地路径如下。

## 1. 替换 Feature List 模板

从 [feature-list.template.md](../templates/feature-list.template.md) 开始，把它映射到公司已有 PRD / feature list 格式。

必须明确：

- 模块和功能点怎么识别
- 业务规则写在哪里
- 交互说明写在哪里
- 异常处理写在哪里
- 验收项写在哪里
- 截图、流程图、状态图如何引用

## 2. 填写公司适配补充

复制 [company-customization.template.md](../templates/company-customization.template.md) 到内部仓库，补齐：

- 测试账号
- App 包名或 Web 域名
- 常用入口路径
- 数据构造方式
- 异常注入方式
- 执行器能力边界

敏感信息留在内部，不要提交到公开仓库。

## 3. 替换输出示例

把 [output-examples.md](../skills/spec-to-testcase/output-examples.md) 里的 Demo 示例换成公司内部最典型的 2 到 3 个业务场景。

建议至少覆盖：

- 一个正向主流程
- 一个反向 / 取消流程
- 一个异常注入流程

## 4. 跑 5 条真实 Case

先不要追求全量生成。选 5 条真实需求，用 AI PHONE 或其他执行器跑完整闭环。

每条都记录：

- 是否执行成功
- 卡在哪一步
- 是写法问题、信息缺口，还是执行器边界
- 应该新增或修改哪条规则

## 5. 把失败原因沉淀回 Rules

只有可复用的失败原因才进入规则。

不要把偶发业务细节写成铁律；也不要把已经多次导致失败的问题继续留在“经验建议”里。

推荐分层：

- Must：不满足就不能输出，例如不可编造、不可占位、单线瀑布、唯一断言、信息缺口必反问。
- Should：强烈建议，例如默认冷启动、emoji 剥离、颜色容差。
- Team-specific：公司自定义，例如 8 级层级输出、特定 PRD 格式、特定造数工具。


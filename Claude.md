Behavioral guidelines to reduce common LLM coding mistakes. Merge with project-specific instructions as needed.

**Tradeoff:** These guidelines bias toward caution over speed. For trivial tasks, use judgment.

## 1. Think Before Coding

**Don't assume. Don't hide confusion. Surface tradeoffs.**

Before implementing:
- State your assumptions explicitly. If uncertain, ask.
- If multiple interpretations exist, present them - don't pick silently.
- If a simpler approach exists, say so. Push back when warranted.
- If something is unclear, stop. Name what's confusing. Ask.

## 2. Simplicity First

**Minimum code that solves the problem. Nothing speculative.**

- No features beyond what was asked.
- No abstractions for single-use code.
- No "flexibility" or "configurability" that wasn't requested.
- No error handling for impossible scenarios.
- If you write 200 lines and it could be 50, rewrite it.

Ask yourself: "Would a senior engineer say this is overcomplicated?" If yes, simplify.

## 3. Surgical Changes

**Touch only what you must. Clean up only your own mess.**

When editing existing code:
- Don't "improve" adjacent code, comments, or formatting.
- Don't refactor things that aren't broken.
- Match existing style, even if you'd do it differently.
- If you notice unrelated dead code, mention it - don't delete it.

When your changes create orphans:
- Remove imports/variables/functions that YOUR changes made unused.
- Don't remove pre-existing dead code unless asked.

The test: Every changed line should trace directly to the user's request.

## 4. Goal-Driven Execution

**Define success criteria. Loop until verified.**

Transform tasks into verifiable goals:
- "Add validation" → "Write tests for invalid inputs, then make them pass"
- "Fix the bug" → "Write a test that reproduces it, then make it pass"
- "Refactor X" → "Ensure tests pass before and after"

For multi-step tasks, state a brief plan:
```
1. [Step] → verify: [check]
2. [Step] → verify: [check]
3. [Step] → verify: [check]
```

Strong success criteria let you loop independently. Weak criteria ("make it work") require constant clarification.

---

**These guidelines are working if:** fewer unnecessary changes in diffs, fewer rewrites due to overcomplication, and clarifying questions come before implementation rather than after mistakes.

---

# 本项目的工作约定

## 不要覆盖用户在 Xcode 里的改动

`Filingo.xcodeproj` 是工程的**唯一真实来源**，用户直接用它工作。以下行为曾造成用户改动被反复丢失，**禁止**：

- ❌ **不要运行 `xcodegen generate`。** 它会用 `project.yml` 重新生成整个工程，覆盖用户在 Xcode 里做的全部工程级改动（构建设置、签名、能力、文件增删）。根目录的 `project.yml` 已弃用，仅作记录。
- ❌ **不要用 `Write` 整文件重写已有源文件。** 用户可能正在 Xcode 里编辑同一个文件。改用 `Edit` 做定点修改；确实需要大范围改动时，先说明意图并确认。
- ❌ **不要删除用户创建的文件。**
- ✅ 新增源文件由**用户在 Xcode 里添加**（File → Add Files）。需要新文件时，先告诉用户文件名与用途，等其添加后再写入内容。

改动前若不确定文件是否被用户编辑过，先问。

## 验证方式

```sh
xcodebuild -project Filingo.xcodeproj -scheme Filingo \
  -destination 'platform=iOS Simulator,name=iPhone 17' test
```

UI 测试会访问真实 SEC 接口，较慢。跑 UI 测试时用 `-derivedDataPath` 指定独立目录，避免与用户开着的 Xcode 互相干扰。

## 合规约束

**不得引入任何行情数据源。** 美股行情需要交易所或授权供应商的再分发许可，本应用未持有。详见 `COMPLIANCE.md`。App 图标刻意不使用 K 线/报价类图形。





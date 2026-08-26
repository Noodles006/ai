Archify
eli5

---
name: git-merge-resolver
description: "将源分支合并到目标分支，解决所有合并冲突，然后推送到新远程分支（不直接推送到目标分支）。 当用户说\"帮我合并\"、\"解决冲突\"、\"merge 到 dev\"、\"这个分支合到那个分支\"、\"合并并推送\"、 \"merge conflict\"、\"分支合并\"等时使用。也适用于用户提供两个分支名并要求合并的场景。 注意：即使用户只提到\"合并\"而没说\"冲突\"，也使用此技能——因为合并过程中随时可能出现冲突。"
metadata:
  version: 4.0.0
---
# Git Merge Resolver

将源分支的改动合并到目标分支，解决冲突，推送到新远程分支 `merge2<target>/<source-branch-name>`。

## 核心原则

**固定使用 `git merge --no-ff` 合并。** 不做分叉检测，不使用 cherry-pick 模式。所有合并统一走 merge 流程，保留合并拓扑，建立祖先关系，避免 MR 虚假冲突。

## 分支方向约束（强制规则）

**环境分支（dev/test/prod/pre/master/main）禁止作为合并目标被合入特性分支。** 这类操作会污染特性分支历史、引入未经验证的环境代码、破坏 MR 审查边界。

### 禁止的方向

| 源分支 | 目标分支 | 是否允许 | 原因 |
|--------|----------|----------|------|
| dev/test/prod/pre | feat/*、fix/*、feature/* | ❌ 禁止 | 环境分支合入特性分支会污染特性历史 |
| master/main | feat/*、fix/* | ❌ 禁止 | 同上，且特性分支应通过 rebase 主动同步而非被合入 |
| 任意环境分支 | 另一环境分支（如 dev→test） | ⚠️ 需用户二次确认 | 环境间合并需人工把关，不可自动执行 |

### 允许的方向

| 源分支 | 目标分支 | 是否允许 |
|--------|----------|----------|
| feat/*、fix/* | dev/test/prod/pre | ✅ 允许（特性 → 环境，标准流程） |
| feat/*、fix/* | master/main | ✅ 允许（特性 → 主干） |
| feat/* | feat/* | ✅ 允许（特性间同步） |

### 执行流程

1. **解析用户意图**后，先判断源/目标分支类型。
2. **若方向被禁止**：立即停止，向用户说明禁止原因，并询问是否确实需要此操作。用户坚持时仍需明确二次确认后才执行，且在执行前再次提示风险。
3. **若方向需确认**（环境→环境）：用 `ask_user_question` 让用户确认后再继续。
4. **若方向允许**：正常进入合并工作流。

### 特性分支识别规则

匹配以下前缀的分支视为特性分支：`feat/`、`fix/`、`feature/`、`chore/`、`refactor/`、`hotfix/`、`bugfix/`、`optimize/`、含 issue 编号的分支（如 `11293-xxx`）。

其余常见长期分支（`dev`、`test`、`prod`、`pre`、`master`、`main`、`release/*`、`develop`）视为环境分支。

## 前置条件

用户需提供：
- **源分支**（source branch）：要合并的分支
- **目标分支**（target branch）：合并到的分支

如果用户没提供完整信息，主动询问。

## 工作流

严格按以下 4 步执行。**不直接在目标分支上 merge，而是先从目标分支创建 `merge2<target>` 分支，在该分支上 merge，保持本地目标分支干净。**

### Step 1: 拉取并检出源分支（用于本地 review）

```bash
git fetch origin
git checkout -b <source-branch> origin/<source-branch>
```

- `git fetch origin` 拉取所有分支最新代码
- 检出源分支到本地，便于 review 改动内容
- 若本地已有同名分支，先 `git checkout <source-branch>` 再 `git pull origin <source-branch>`

### Step 2: 本地 review 源分支改动

读取源分支涉及的文件，了解改动内容。可用：

```bash
git log --oneline origin/<target-branch>..origin/<source-branch>
git diff origin/<target-branch>...origin/<source-branch> --stat
```

此步骤为理解改动，不修改代码。

### Step 3: 创建 merge2 分支并执行 merge --no-ff

**关键：不直接在目标分支上 merge，先创建 merge2 分支。**

```bash
git checkout <target-branch>
git pull origin <target-branch>
git checkout -b merge2<target>/<source-branch-name>
git merge --no-ff origin/<source-branch> --no-edit
```

分支名规则：取源分支名中 `/` 后的部分（如 `feat/11293-权限管理优化` → `11293-权限管理优化`），组成 `merge2<target>/<提取部分>`，如 `merge2dev/11293-权限管理优化`。

若该分支已存在，先删除：`git branch -D merge2<target>/<提取部分>`，再创建。

- `--no-ff` 强制产生 merge commit，保留合并拓扑，建立祖先关系
- `--no-edit` 使用默认 commit message

#### 无冲突

merge 自动完成，跳到 Step 4。

#### 有冲突

当 `git merge` 报 `CONFLICT` 时：

1. **列出冲突文件**：
   ```bash
   git diff --check
   ```

2. **逐个读取冲突文件**，找到 `<<<<<<<` / `=======` / `>>>>>>>` 标记。

3. **分析并解决**（详见下方「冲突解决原则」）：
   - `<<<<<<< HEAD` 到 `=======` 之间是目标分支（merge2 分支，基于 target）的内容
   - `=======` 到 `>>>>>>> origin/<source-branch>` 之间是源分支的内容
   - **核心策略：分析判断，保留更完整/更新的版本，而非机械两边都保留。**

4. **验证冲突全部清除**：
   ```bash
   git diff --check
   ```
   无输出 = 全部解决。

5. **提交 merge**：
   ```bash
   git add <所有冲突文件>
   git commit --no-edit
   ```

#### 放弃 merge

```bash
git merge --abort
```

#### 验证祖先关系（可选）

确认源分支已包含在 merge 分支历史中：

```bash
git merge-base --is-ancestor origin/<source-branch> HEAD && echo "OK" || echo "FAIL"
```

应输出 `OK`。若 `FAIL`，说明 merge 未成功，需排查。

### Step 4: 推送 merge2 分支到远程

```bash
git push origin merge2<target>/<source-branch-name>
```

若远程已有同名分支（旧版本），强制推送覆盖：

```bash
git push origin merge2<target>/<source-branch-name> --force-with-lease
```

推送后告知用户可创建 MR 的链接。

---

## 冲突解决原则

**核心策略：分析判断，保留更完整/更新的版本，而非机械地两边都保留。** 目标分支（HEAD）通常是最新主干，包含更完整的功能实现；源分支是特性分支，可能只是早期版本。冲突解决时优先识别哪一侧是"完整版"，保留它，丢弃另一侧的"基础版"。

### 判断方法

对每个冲突块，逐块分析两侧内容差异：

1. **识别完整度差异**：一侧有额外功能（如多了图标、多了 state、多了渲染逻辑、多了 props），另一侧是子集 → 保留完整侧（通常是 HEAD）。
2. **识别版本新旧**：一侧是新格式/新 API，另一侧是旧格式 → 保留新侧。
3. **真正互补的情况**：两侧各自有对方没有的独立内容（如不同的 import、不同的字段）→ 两边都保留，去重排序。

### 按场景的策略

- **import 语句冲突**：取两侧并集，去重排序。若一侧是另一侧的子集，保留超集侧。
- **state/字段声明冲突**：分析功能完整度。若 HEAD 有 `groupMembersMap` 等额外 state，源分支只有 `searchValue`，且 `searchValue` 在 HEAD 其他位置已声明 → 保留 HEAD，丢弃源分支重复声明。
- **组件 props / JSX 渲染冲突**：识别哪侧是完整功能版。如 HEAD 有 `renderSourceOption`、`resolveTargetValues`、`renderTargetOptionBatch/Normal` 完整配置，源分支只有基础 `optionFilter`、`renderTargetOption` → 保留 HEAD 完整版。
- **方法/函数冲突**：优先保留目标分支逻辑，再合并源分支新增逻辑；复杂情况暂停询问用户。
- **配置文件冲突**：以目标分支（HEAD）为基础，补充源分支独有的字段。
- **package.json 的 packageManager 字段冲突**：保留目标分支版本（HEAD），源分支通常无此字段。
- **dataStore / pin 等缓存配置冲突**：保留目标分支的新版格式（如 `{ key: location.pathname }`），源分支旧版（如 `true`）丢弃。
- **无法判断的冲突**：向用户展示冲突内容，让用户决策。

### 实际案例参考

`feat/11293-权限管理优化` → `dev` 合并中，`DimValueSelectModal.tsx` 有 3 处冲突：

| 冲突位置 | HEAD（dev） | 源分支 | 解决 |
|---------|-------------|--------|------|
| import | `Modal, TransferV2, Checkbox, Spin, Tooltip` + `LayersFilled, InfoCircle, Close` | 仅 `Modal, TransferV2, Checkbox, Spin` | 保留 HEAD（多 Tooltip 和图标，是完整版） |
| state 声明 | `groupMembersMap` | `searchValue`（与下方重复） | 保留 HEAD 的 `groupMembersMap`，丢弃源分支重复的 `searchValue` |
| TransferV2 props | 完整配置（optionFilter as any、renderSourceOption、resolveTargetValues、renderTargetOptionBatch/Normal） | 基础配置（optionFilter、renderTargetOption） | 保留 HEAD 完整版 |

`RoleCard.tsx` 1 处冲突：HEAD 多 `LayersFilled` 图标（维度组 Tag 用），保留 HEAD。

## 批量冲突处理技巧

当大量文件存在相同模式的冲突（如 25 个文件都是同一字段的新旧版本冲突）：

1. **先抽样检查 1-2 个文件**，确认冲突模式一致
2. **用 perl 批量处理**（保留 HEAD 侧）：
   ```bash
   for f in $(git diff --name-only --diff-filter=U); do
     perl -i -0pe 's/<<<<<<< HEAD\n(.*?\n)=======\n.*?\n>>>>>>> [0-9a-f]+.*?\n/$1/s' "$f"
   done
   ```
   注意：正则中 `.*?\n` 匹配源分支侧内容，需根据实际调整。
3. **验证**：`git diff --check` 无输出 = 全部解决
4. **抽查**：读取 1-2 个文件确认结果正确

## 安全检查

解决每个冲突文件后，必须确认：
- 代码语法完整（括号匹配、分号不缺）
- import 语句没有重复
- 没有残留的 `<<<<<<<` / `=======` / `>>>>>>>` 标记
- JSON 文件用 `node -e "JSON.parse(...)"` 验证语法

## 常见陷阱

### 陷阱 1: 直接在目标分支上 merge 污染本地目标分支

**症状：** `git checkout dev && git merge --no-ff <source>` 后本地 dev 被 merge 污染，需 reset 还原。

**解决：** 先从 target 创建 `merge2<target>` 分支，在该分支上 merge，保持本地 target 干净。

### 陷阱 2: 空提交堆积

**症状：** merge 后某些文件无实际改动却显示冲突。

**原因：** 源分支的某些改动目标分支已有。

**解决：** 解决冲突后正常 `git commit --no-edit` 即可，merge 模式不会产生空提交问题。

### 陷阱 3: 远程已有同名 merge2 分支

**症状：** `git push` 报 `! [rejected]` 非快进。

**解决：** 用 `--force-with-lease` 强制推送覆盖旧版本。

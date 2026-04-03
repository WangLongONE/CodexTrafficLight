# 私有 GitLab 一句话 MR + 合并后钉钉通知 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 提供 `mr` 命令在本地一句话自动 push 并创建 GitLab MR，并在 MR merged 后由服务端自动推送钉钉群机器人通知。

**Architecture:** 使用 Node.js + TypeScript 单仓两应用结构：`apps/mr-cli` 负责命令解析与 GitLab MR 创建，`apps/merge-notifier` 负责 GitLab webhook 事件消费、幂等与钉钉通知。公共逻辑放在 `packages/common`，测试框架统一 `vitest`。

**Tech Stack:** Node.js 20, TypeScript 5, Vitest, Commander, Fastify, Zod, Undici

---

## File Structure

- `package.json`：workspace 管理与脚本入口。
- `tsconfig.base.json`：TS 基础配置。
- `vitest.config.ts`：统一测试配置。
- `apps/mr-cli/src/index.ts`：CLI 入口。
- `apps/mr-cli/src/command.ts`：命令解析与执行流。
- `apps/mr-cli/src/git.ts`：Git 命令封装。
- `apps/mr-cli/src/gitlab.ts`：GitLab API 封装。
- `apps/mr-cli/src/config.ts`：配置读取。
- `apps/mr-cli/test/*.test.ts`：CLI 单测。
- `apps/merge-notifier/src/server.ts`：Fastify 服务入口。
- `apps/merge-notifier/src/handler.ts`：webhook 处理逻辑。
- `apps/merge-notifier/src/dingtalk.ts`：钉钉发送。
- `apps/merge-notifier/src/recipients.ts`：手机号汇总与去重。
- `apps/merge-notifier/src/idempotency.ts`：幂等存储。
- `apps/merge-notifier/test/*.test.ts`：notifier 单测。
- `packages/common/src/types.ts`：共享类型定义。
- `.env.example`：环境变量示例。
- `.gitignore`：忽略规则。

### Task 1: 初始化仓库与测试基线

**Files:**
- Create: `package.json`
- Create: `tsconfig.base.json`
- Create: `vitest.config.ts`
- Create: `apps/mr-cli/package.json`
- Create: `apps/merge-notifier/package.json`
- Create: `packages/common/package.json`
- Modify: `.gitignore`

- [ ] **Step 1: 写失败测试，验证 workspace 结构与测试命令可执行**

```ts
// apps/mr-cli/test/smoke.test.ts
import { describe, expect, it } from 'vitest';

describe('workspace smoke', () => {
  it('runs test runner', () => {
    expect(true).toBe(true);
  });
});
```

- [ ] **Step 2: 运行测试，确认当前失败（缺少依赖/脚本）**

Run: `npm test`
Expected: FAIL with "Missing script: test" or workspace config error

- [ ] **Step 3: 写最小实现（workspace 与 vitest 配置）**

```json
// package.json
{
  "name": "gitlab-mr-dingtalk",
  "private": true,
  "workspaces": ["apps/*", "packages/*"],
  "scripts": {
    "test": "vitest run",
    "build": "tsc -b"
  },
  "devDependencies": {
    "typescript": "^5.6.0",
    "vitest": "^2.1.0"
  }
}
```

```json
// apps/mr-cli/package.json
{
  "name": "mr-cli",
  "version": "0.1.0",
  "bin": {
    "mr": "dist/index.js"
  },
  "type": "module"
}
```

```json
// apps/merge-notifier/package.json
{
  "name": "merge-notifier",
  "version": "0.1.0",
  "type": "module"
}
```

```json
// packages/common/package.json
{
  "name": "@mr/common",
  "version": "0.1.0",
  "type": "module"
}
```

- [ ] **Step 4: 运行测试，确认通过**

Run: `npm test`
Expected: PASS with 1 passed test

- [ ] **Step 5: Commit**

```bash
git add package.json tsconfig.base.json vitest.config.ts apps packages .gitignore
git commit -m "chore: initialize monorepo and test baseline"
```

### Task 2: CLI 命令解析与配置读取

**Files:**
- Create: `apps/mr-cli/src/index.ts`
- Create: `apps/mr-cli/src/command.ts`
- Create: `apps/mr-cli/src/config.ts`
- Create: `apps/mr-cli/test/command.test.ts`

- [ ] **Step 1: 写失败测试（`mr a 到 b` 解析）**

```ts
import { describe, expect, it } from 'vitest';
import { parseCommand } from '../src/command';

describe('parseCommand', () => {
  it('parses source and target branches', () => {
    const parsed = parseCommand(['feat/login', '到', 'main']);
    expect(parsed).toEqual({ source: 'feat/login', target: 'main', title: undefined, desc: undefined });
  });
});
```

- [ ] **Step 2: 运行测试确认失败**

Run: `npm test -- apps/mr-cli/test/command.test.ts`
Expected: FAIL with "Cannot find module '../src/command'"

- [ ] **Step 3: 写最小实现（解析+配置）**

```ts
// apps/mr-cli/src/command.ts
export type CommandInput = { source: string; target: string; title?: string; desc?: string };

export function parseCommand(argv: string[]): CommandInput {
  if (argv.length < 3 || argv[1] !== '到') {
    throw new Error('用法: mr <源分支> 到 <目标分支> [--title 标题] [--desc 描述]');
  }
  return { source: argv[0], target: argv[2] };
}
```

```ts
// apps/mr-cli/src/config.ts
export type CliConfig = {
  gitlabBaseUrl: string;
  gitlabToken: string;
  projectId: string;
};

export function readConfig(env = process.env): CliConfig {
  const gitlabBaseUrl = env.GITLAB_BASE_URL ?? '';
  const gitlabToken = env.GITLAB_TOKEN ?? '';
  const projectId = env.GITLAB_PROJECT_ID ?? '';
  if (!gitlabBaseUrl || !gitlabToken || !projectId) {
    throw new Error('缺少 GITLAB_BASE_URL/GITLAB_TOKEN/GITLAB_PROJECT_ID');
  }
  return { gitlabBaseUrl, gitlabToken, projectId };
}
```

- [ ] **Step 4: 运行测试确认通过**

Run: `npm test -- apps/mr-cli/test/command.test.ts`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add apps/mr-cli/src/index.ts apps/mr-cli/src/command.ts apps/mr-cli/src/config.ts apps/mr-cli/test/command.test.ts
git commit -m "feat(cli): add command parser and config loader"
```

### Task 3: CLI Git 操作封装（push）

**Files:**
- Create: `apps/mr-cli/src/git.ts`
- Create: `apps/mr-cli/test/git.test.ts`
- Modify: `apps/mr-cli/src/command.ts`

- [ ] **Step 1: 写失败测试（会调用 push）**

```ts
import { describe, expect, it, vi } from 'vitest';
import { ensurePushed } from '../src/git';

describe('ensurePushed', () => {
  it('runs git push -u origin <source>', async () => {
    const run = vi.fn().mockResolvedValue({ code: 0, stdout: '', stderr: '' });
    await ensurePushed('feat/login', run);
    expect(run).toHaveBeenCalledWith(['git', 'push', '-u', 'origin', 'feat/login']);
  });
});
```

- [ ] **Step 2: 运行测试确认失败**

Run: `npm test -- apps/mr-cli/test/git.test.ts`
Expected: FAIL with module not found

- [ ] **Step 3: 写最小实现（Git runner）**

```ts
// apps/mr-cli/src/git.ts
export type RunCmd = (cmd: string[]) => Promise<{ code: number; stdout: string; stderr: string }>;

export async function ensurePushed(source: string, run: RunCmd): Promise<void> {
  const result = await run(['git', 'push', '-u', 'origin', source]);
  if (result.code !== 0) {
    throw new Error(`git push failed: ${result.stderr}`);
  }
}
```

- [ ] **Step 4: 运行测试确认通过**

Run: `npm test -- apps/mr-cli/test/git.test.ts`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add apps/mr-cli/src/git.ts apps/mr-cli/src/command.ts apps/mr-cli/test/git.test.ts
git commit -m "feat(cli): add git push wrapper"
```

### Task 4: GitLab MR 查询与创建（避免重复 MR）

**Files:**
- Create: `apps/mr-cli/src/gitlab.ts`
- Create: `apps/mr-cli/test/gitlab.test.ts`
- Modify: `apps/mr-cli/src/command.ts`

- [ ] **Step 1: 写失败测试（已有 opened MR 返回已有 URL）**

```ts
import { describe, expect, it, vi } from 'vitest';
import { findOrCreateMr } from '../src/gitlab';

describe('findOrCreateMr', () => {
  it('returns existing opened MR url when matched', async () => {
    const api = {
      listOpenedMrs: vi.fn().mockResolvedValue([{ iid: 12, web_url: 'https://gitlab.local/mr/12' }]),
      createMr: vi.fn()
    };
    const url = await findOrCreateMr(api as any, 'feat/login', 'main', 'feat/login -> main');
    expect(url).toBe('https://gitlab.local/mr/12');
    expect(api.createMr).not.toHaveBeenCalled();
  });
});
```

- [ ] **Step 2: 运行测试确认失败**

Run: `npm test -- apps/mr-cli/test/gitlab.test.ts`
Expected: FAIL with module not found

- [ ] **Step 3: 写最小实现（list + create）**

```ts
// apps/mr-cli/src/gitlab.ts
export type Mr = { iid: number; web_url: string };
export type GitlabApi = {
  listOpenedMrs: (source: string, target: string) => Promise<Mr[]>;
  createMr: (source: string, target: string, title: string, desc?: string) => Promise<Mr>;
};

export async function findOrCreateMr(
  api: GitlabApi,
  source: string,
  target: string,
  title: string,
  desc?: string
): Promise<string> {
  const existing = await api.listOpenedMrs(source, target);
  if (existing.length > 0) {
    return existing[0].web_url;
  }
  const created = await api.createMr(source, target, title, desc);
  return created.web_url;
}
```

- [ ] **Step 4: 运行测试确认通过**

Run: `npm test -- apps/mr-cli/test/gitlab.test.ts`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add apps/mr-cli/src/gitlab.ts apps/mr-cli/src/command.ts apps/mr-cli/test/gitlab.test.ts
git commit -m "feat(cli): add find-or-create merge request flow"
```

### Task 5: Notifier 基础 webhook（仅 merged 触发）

**Files:**
- Create: `apps/merge-notifier/src/server.ts`
- Create: `apps/merge-notifier/src/handler.ts`
- Create: `apps/merge-notifier/test/handler.test.ts`
- Create: `packages/common/src/types.ts`

- [ ] **Step 1: 写失败测试（非 merged 忽略，merged 触发）**

```ts
import { describe, expect, it, vi } from 'vitest';
import { handleMrEvent } from '../src/handler';

describe('handleMrEvent', () => {
  it('ignores non-merged event', async () => {
    const send = vi.fn();
    const result = await handleMrEvent({ object_kind: 'merge_request', object_attributes: { state: 'opened' } } as any, send);
    expect(result).toBe('ignored');
    expect(send).not.toHaveBeenCalled();
  });
});
```

- [ ] **Step 2: 运行测试确认失败**

Run: `npm test -- apps/merge-notifier/test/handler.test.ts`
Expected: FAIL with module not found

- [ ] **Step 3: 写最小实现（state=merged 过滤）**

```ts
// apps/merge-notifier/src/handler.ts
export async function handleMrEvent(payload: any, send: (text: string, atMobiles: string[]) => Promise<void>): Promise<'ignored' | 'sent'> {
  if (payload?.object_kind !== 'merge_request') return 'ignored';
  if (payload?.object_attributes?.state !== 'merged') return 'ignored';

  const text = `MR 已合并: ${payload.object_attributes.title}\n${payload.object_attributes.url}`;
  await send(text, []);
  return 'sent';
}
```

- [ ] **Step 4: 运行测试确认通过**

Run: `npm test -- apps/merge-notifier/test/handler.test.ts`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add apps/merge-notifier/src/server.ts apps/merge-notifier/src/handler.ts apps/merge-notifier/test/handler.test.ts packages/common/src/types.ts
git commit -m "feat(notifier): handle merged merge-request webhook"
```

### Task 6: 手机号聚合、幂等、防重与钉钉重试

**Files:**
- Create: `apps/merge-notifier/src/recipients.ts`
- Create: `apps/merge-notifier/src/idempotency.ts`
- Create: `apps/merge-notifier/src/dingtalk.ts`
- Create: `apps/merge-notifier/test/recipients.test.ts`
- Create: `apps/merge-notifier/test/idempotency.test.ts`
- Create: `apps/merge-notifier/test/dingtalk.test.ts`
- Modify: `apps/merge-notifier/src/handler.ts`

- [ ] **Step 1: 写失败测试（固定+动态手机号合并去重 + 幂等）**

```ts
import { describe, expect, it } from 'vitest';
import { mergeMobiles } from '../src/recipients';

describe('mergeMobiles', () => {
  it('deduplicates fixed and dynamic mobiles', () => {
    const result = mergeMobiles(['13800000001', '13800000002'], ['13800000002', '13800000003']);
    expect(result).toEqual(['13800000001', '13800000002', '13800000003']);
  });
});
```

- [ ] **Step 2: 运行测试确认失败**

Run: `npm test -- apps/merge-notifier/test/recipients.test.ts`
Expected: FAIL with module not found

- [ ] **Step 3: 写最小实现（聚合、幂等键、重试）**

```ts
// apps/merge-notifier/src/recipients.ts
export function mergeMobiles(fixed: string[], dynamic: string[]): string[] {
  return [...new Set([...fixed, ...dynamic].filter(Boolean))];
}
```

```ts
// apps/merge-notifier/src/idempotency.ts
const seen = new Set<string>();

export function makeEventKey(projectId: number, iid: number, mergedAt: string): string {
  return `${projectId}:${iid}:${mergedAt}`;
}

export function shouldProcess(key: string): boolean {
  if (seen.has(key)) return false;
  seen.add(key);
  return true;
}
```

```ts
// apps/merge-notifier/src/dingtalk.ts
export async function sendWithRetry(
  sendOnce: () => Promise<void>,
  sleep: (ms: number) => Promise<void>
): Promise<void> {
  const delays = [0, 2000, 4000];
  let lastError: unknown;
  for (const delay of delays) {
    if (delay > 0) await sleep(delay);
    try {
      await sendOnce();
      return;
    } catch (err) {
      lastError = err;
    }
  }
  throw lastError;
}
```

- [ ] **Step 4: 运行测试确认通过**

Run: `npm test -- apps/merge-notifier/test/recipients.test.ts apps/merge-notifier/test/idempotency.test.ts apps/merge-notifier/test/dingtalk.test.ts`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add apps/merge-notifier/src/recipients.ts apps/merge-notifier/src/idempotency.ts apps/merge-notifier/src/dingtalk.ts apps/merge-notifier/src/handler.ts apps/merge-notifier/test
git commit -m "feat(notifier): add recipients merge, idempotency, and dingtalk retry"
```

### Task 7: 文档、环境变量与联调验收

**Files:**
- Create: `.env.example`
- Create: `README.md`
- Modify: `apps/mr-cli/src/index.ts`
- Modify: `apps/merge-notifier/src/server.ts`

- [ ] **Step 1: 写失败测试（缺失关键环境变量时报错）**

```ts
import { describe, expect, it } from 'vitest';
import { readConfig } from '../src/config';

describe('readConfig', () => {
  it('throws when required env missing', () => {
    expect(() => readConfig({} as any)).toThrow(/缺少/);
  });
});
```

- [ ] **Step 2: 运行测试确认失败（若当前未抛错）**

Run: `npm test -- apps/mr-cli/test/command.test.ts`
Expected: FAIL if no required-env guard exists

- [ ] **Step 3: 写最小实现（示例配置与使用说明）**

```dotenv
# .env.example
GITLAB_BASE_URL=https://gitlab.example.com
GITLAB_TOKEN=glpat-xxxx
GITLAB_PROJECT_ID=123
GITLAB_WEBHOOK_TOKEN=your-webhook-token
DINGTALK_WEBHOOK=https://oapi.dingtalk.com/robot/send?access_token=xxx
DINGTALK_SECRET=
NOTIFY_FIXED_MOBILES=13800000001,13800000002
```

```md
# README.md
## CLI
mr feat/login 到 main

## notifier
npm run dev:notifier
# configure GitLab Merge Request Hook -> /webhooks/gitlab
```

- [ ] **Step 4: 运行全量测试确认通过**

Run: `npm test`
Expected: PASS with all suites green

- [ ] **Step 5: Commit**

```bash
git add .env.example README.md apps/mr-cli apps/merge-notifier
git commit -m "docs: add usage guide and env examples"
```

## Self-Review

- Spec coverage:
  - 一句话命令创建 MR：Task 2-4 覆盖。
  - merged 后通知：Task 5 覆盖。
  - 固定+动态手机号合并：Task 6 覆盖。
  - 幂等与重试：Task 6 覆盖。
  - 产物与使用说明：Task 7 覆盖。
- Placeholder scan:
  - 已检查，无 TBD/TODO/“后续补充”等占位语句。
- Type consistency:
  - `source/target/title/desc` 在 CLI 模块内一致。
  - `projectId/iid/mergedAt` 幂等键字段在 notifier 内一致。

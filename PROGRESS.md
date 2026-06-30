# Services SDK 0.6.0 Upgrade Progress

本文档把 `services/` SDK 0.6.0 升级拆成可独立执行、独立验收的任务清单。任务按依赖顺序排列；标记为“可并行”的子任务可以在同一父任务内用 subagent 并行推进，但 subagent 并发度最高不超过 5。

## 文档索引

- 技术方案：[docs/spec/services-sdk-0-6-upgrade-spec.md](docs/spec/services-sdk-0-6-upgrade-spec.md)
- 实施计划：[docs/plan/services-sdk-0-6-upgrade-implementation-plan.md](docs/plan/services-sdk-0-6-upgrade-implementation-plan.md)
- Harness：[AGENTS.md](AGENTS.md)
- Task 工作流：[Taskfile.yml](Taskfile.yml)
- Services 质量线：[docs/design/technical/services-package-quality.md](docs/design/technical/services-package-quality.md)
- SDK 设计：[docs/design/technical/js-sdk.md](docs/design/technical/js-sdk.md)
- 发布和生成物策略：[docs/design/technical/release.md](docs/design/technical/release.md)
- CI：[.github/workflows/ci.yml](.github/workflows/ci.yml)

## 执行规则

- [ ] 每个任务完成时必须同时完成对应测试方案和验收标准。
- [ ] 不跨阶段提前合并依赖未满足的功能；阶段 2 后项目必须保持 services 可验证。
- [ ] 不修改 proto、schema、service name、bin、handler key、runtime mode 或上游业务字段。
- [ ] 不提交 `services/package-lock.json`、`node_modules/`、pack artifact、日志、coverage、`.env` 或 secret。
- [ ] helper 迁移只做语义等价或已由测试覆盖的变更；遇到错误码、message、details 或 payload shape 差异时停止扩大批次。
- [ ] 每个任务合并前至少运行该任务要求的最小测试；阶段性收口时运行 harness 定义的完整门禁。
- [ ] 每个任务完成后必须按 `状态`、`变更`、`验证`、`审计与例外`、`下一目标` 更新完成总结。

## 1. 基线确认和变更清单

参考文档：[实施计划 阶段 1](docs/plan/services-sdk-0-6-upgrade-implementation-plan.md#阶段-1基线确认和变更清单)

- [x] 1.1 确认 SDK 0.6.0 和 services 当前依赖面
  - 依赖：无。
  - 工作内容：
    - 确认 npmjs `@chaitin-ai/octobus-sdk@latest` 为 `0.6.0`，且依赖、engine、bin、types 与 spec 一致。
    - 统计 `services/package.json` 和 `services/*/package.json` 中 `@chaitin-ai/octobus-sdk` `^0.5.0` 命中数。
    - 确认 `services/package-lock.json`、`services/node_modules/` 和 pack artifact 不应提交。
    - 记录当前 `git status --short`，避免后续误回滚无关变更。
  - 可并行子任务：
    - [x] 可并行：npm registry 元数据确认。
    - [x] 可并行：services dependency 命中统计。
    - [x] 可并行：生成物和 git 状态审计。
  - 测试方案：
    - `npm view @chaitin-ai/octobus-sdk@latest version dependencies engines main types bin --json`
    - `rg '"@chaitin-ai/octobus-sdk": "\\^0\\.5\\.0"' services/package.json services/*/package.json`
    - `git status --short`
  - 验收标准：
    - 确认目标依赖为 `^0.6.0`，不使用 `latest`。
    - 确认需要更新的 dependency 声明范围和相关文档/fixture 范围。
    - 未发现需要提交的 lockfile 或 node_modules。
  - 完成总结：
    - 状态：已完成。确认 SDK 0.6.0 发布事实、services 当前依赖面和本地生成物边界，未修改业务源码。
    - 变更：
      - 更新本任务 checkbox 和完成证据。
      - 确认目标依赖应固定写为 `^0.6.0`，不使用 `latest`。
      - 确认当前需要升级的 package dependency 声明为 51 处：`services/package.json` 1 处和 50 个 service root `package.json`。
      - 确认 service root dependency 形态为 1 个阿里云 SDK + OctoBus SDK、44 个 OctoBus SDK + `undici`、5 个仅 OctoBus SDK。
    - 验证：
      - `npm view @chaitin-ai/octobus-sdk@latest version dependencies engines main types bin --json`：返回 `version` 为 `0.6.0`，`engines.node` 为 `>=20`，`main` 为 `dist/index.js`，`types` 为 `dist/index.d.ts`，`bin.octobus-sdk` 为 `dist/cli.js`，依赖包含 `undici`。
      - `rg '"@chaitin-ai/octobus-sdk": "\\^0\\.5\\.0"' services/package.json services/*/package.json`：命中 51 处。
      - Node dependency 统计脚本：确认 51 个 package JSON 均声明 `@chaitin-ai/octobus-sdk`，当前版本集合仅为 `^0.5.0`。
      - `git check-ignore -v services/package-lock.json services/node_modules services/example.tgz services/foo.tar.gz services/foo.zip services/foo.log services/coverage 2>/dev/null || true`：确认 `services/package-lock.json`、`services/node_modules`、日志和 coverage 由 ignore 规则覆盖；`.tgz`、`.tar.gz`、`.zip` 不在通用 ignore 中，后续必须通过 pack/污染审计避免提交。
      - `git ls-files services/package-lock.json services/node_modules 'services/*.tgz' 'services/*.tar.gz' 'services/*.zip' 'services/*.log' services/coverage`：无 tracked 输出。
      - `find services -maxdepth 2 \( -name 'package-lock.json' -o -name 'node_modules' -o -name '*.tgz' -o -name '*.tar.gz' -o -name '*.zip' -o -name '*.log' -o -name '.env' -o -name 'coverage' \) -print | sort`：本地存在 `services/node_modules` 和 `services/package-lock.json`，均为 ignored 生成物，不进入提交。
      - `git status --short`：任务开始时仅有已 staged 的 `PROGRESS.md`、`docs/plan/services-sdk-0-6-upgrade-implementation-plan.md`、`docs/spec/services-sdk-0-6-upgrade-spec.md`。
    - 审计与例外：
      - 未修改 `services/` package、proto、schema、service name、bin、handler key、runtime mode、dispatcher mapping 或业务源码。
      - 当前 `services/package-lock.json` 和 `services/node_modules` 在工作区存在但被 ignore；后续任务运行前仍需持续审计并避免纳入提交。
      - `.tgz`、`.tar.gz`、`.zip` pack artifact 当前未发现 tracked 或 untracked 待提交文件，但不依赖 ignore 规则保护，阶段 2/6/7 必须继续检查。
    - 下一目标：任务 2.1。

## 2. 统一 SDK Dependency 版本

参考文档：[实施计划 阶段 2](docs/plan/services-sdk-0-6-upgrade-implementation-plan.md#阶段-2统一-sdk-dependency-版本)

- [x] 2.1 批量更新 services package SDK 版本
  - 依赖：任务 1.1。
  - 工作内容：
    - 将 `services/package.json` 的 `dependencies["@chaitin-ai/octobus-sdk"]` 从 `^0.5.0` 改为 `^0.6.0`。
    - 将所有 `services/*/package.json` 的直接 SDK dependency 从 `^0.5.0` 改为 `^0.6.0`。
    - 保留每个 package 文件的既有依赖顺序、缩进和其他依赖版本。
    - 保留已存在的 `undici` 直接依赖和根 `bundledDependencies`。
  - 可并行子任务：
    - [x] 可并行：root `services/package.json` 更新。
    - [x] 可并行：50 个 service root `package.json` 更新，可按目录分片。
  - 测试方案：
    - `rg '"@chaitin-ai/octobus-sdk": "\\^0\\.5\\.0"' services/package.json services/*/package.json`
    - `node -e 'const fs=require("fs"),path=require("path"); const files=["services/package.json",...fs.readdirSync("services").map(d=>path.join("services",d,"package.json")).filter(fs.existsSync)]; for (const f of files){const p=JSON.parse(fs.readFileSync(f,"utf8")); if(p.dependencies?.["@chaitin-ai/octobus-sdk"]!=="^0.6.0") throw new Error(f);}'`
  - 验收标准：
    - 51 处 package dependency 均为 `^0.6.0`。
    - 未修改 proto、schema、service name、bin、handler key 或业务源码。
    - 没有新增 tracked lockfile 或 node_modules。
  - 完成总结：
    - 状态：已完成。完成纯 dependency 声明升级，未修改业务源码或 runtime 契约文件。
    - 变更：
      - `services/package.json` 中 `dependencies["@chaitin-ai/octobus-sdk"]` 从 `^0.5.0` 更新为 `^0.6.0`。
      - 50 个 `services/*/package.json` 中直接 SDK dependency 从 `^0.5.0` 更新为 `^0.6.0`。
      - 保留根 `bundledDependencies` 为 `@alicloud/swas-open20200601`、`@chaitin-ai/octobus-sdk`、`commander`、`undici`。
      - 未修改 proto、schema、service name、bin、handler key、dispatcher mapping、runtime mode 或业务源码。
    - 验证：
      - `rg -l '"@chaitin-ai/octobus-sdk": "\\^0\\.5\\.0"' services/package.json services/*/package.json | wc -l`：任务开始前确认命中 51 个 package 文件。
      - `rg '"@chaitin-ai/octobus-sdk": "\\^0\\.5\\.0"' services/package.json services/*/package.json || true`：升级后无输出。
      - `node -e 'const fs=require("fs"),path=require("path"); const files=["services/package.json",...fs.readdirSync("services").map(d=>path.join("services",d,"package.json")).filter(fs.existsSync)]; let count=0; for (const f of files){const p=JSON.parse(fs.readFileSync(f,"utf8")); if(p.dependencies?.["@chaitin-ai/octobus-sdk"]!=="^0.6.0") throw new Error(f); count++;} console.log(`checked ${count} package files`);'`：输出 `checked 51 package files`。
      - 根 `services/package.json` 审计脚本：确认 SDK dependency 为 `^0.6.0`，`bundledDependencies` 仍包含原 4 项运行时依赖。
      - `git diff --name-only`：仅 51 个 package JSON 发生变更。
      - `git diff --stat`：51 个文件各 1 行版本号替换，总计 51 insertions、51 deletions。
      - `git ls-files --others --exclude-standard`：无输出。
      - `find services -maxdepth 2 \( -name '*.tgz' -o -name '*.tar.gz' -o -name '*.zip' -o -name '*.log' -o -name '.env' -o -name 'coverage' \) -print | sort`：无输出。
    - 审计与例外：
      - 本任务只覆盖 dependency 声明；`services/tests/validate-service-package.test.mjs`、service README 和设计文档示例仍由任务 2.2 更新。
      - 本地 ignored 的 `services/package-lock.json` 和 `services/node_modules` 未进入 `git status` 或提交范围。
    - 下一目标：任务 2.2。

- [x] 2.2 更新依赖版本相关 fixture 和文档示例
  - 依赖：任务 2.1。
  - 工作内容：
    - 更新 `services/tests/validate-service-package.test.mjs` 中测试 fixture 的 SDK 版本字符串为 `^0.6.0`。
    - 更新 `services/first__epss-v1/README.md` 中 SDK 版本说明为 `^0.6.0`。
    - 更新 `docs/design/technical/multi-service-npm-package.md` 和 `docs/design/technical/service-package.md` 中 services package 示例依赖为 `^0.6.0`。
  - 可并行子任务：
    - [x] 可并行：测试 fixture 更新。
    - [x] 可并行：service README 更新。
    - [x] 可并行：设计文档示例更新。
  - 测试方案：
    - `rg '\\^0\\.5\\.0|0\\.5\\.0' services docs/design/technical/multi-service-npm-package.md docs/design/technical/service-package.md`
  - 验收标准：
    - services 和相关设计文档不再引用 SDK 0.5.0。
    - `examples/*` 未被修改；示例升级不在首版范围内。
  - 完成总结：
    - 状态：已完成。完成 SDK 版本相关 fixture、service README 和设计文档示例同步。
    - 变更：
      - `services/tests/validate-service-package.test.mjs` 中 4 个测试 fixture 的 SDK dependency 改为 `^0.6.0`。
      - `services/first__epss-v1/README.md` 中 SDK 版本说明改为 `^0.6.0`。
      - `docs/design/technical/multi-service-npm-package.md` 中 services package 示例依赖改为 `^0.6.0`。
      - `docs/design/technical/service-package.md` 中 JavaScript service package 依赖基线和 SDK npmjs 分发示例改为 `^0.6.0`。
    - 验证：
      - `rg -n '\\^0\\.5\\.0|0\\.5\\.0' services docs/design/technical/multi-service-npm-package.md docs/design/technical/service-package.md || true`：无输出。
      - `git diff --name-only -- examples`：无输出，确认未修改 `examples/*`。
      - `git diff --name-only`：仅包含两份设计文档、`services/first__epss-v1/README.md` 和 `services/tests/validate-service-package.test.mjs`。
    - 审计与例外：
      - 本任务没有运行 services validate/test/pack check；完整纯依赖升级门禁由任务 2.3 执行。
      - 未修改 package import、proto、schema、service name、bin、handler key、runtime mode 或业务源码。
    - 下一目标：任务 2.3。

- [ ] 2.3 跑纯依赖升级 services 门禁
  - 依赖：任务 2.1、任务 2.2。
  - 工作内容：
    - 清理不应提交的 `services/package-lock.json`、`services/node_modules/`、pack artifact 和日志。
    - 运行 services 结构、测试和 pack dry-run 门禁。
    - 如果仅依赖升级导致测试失败，停止后续 helper 迁移并定位 SDK 0.6.0 兼容性问题。
  - 可并行子任务：
    - [ ] 可并行：运行 `npm run validate`。
    - [ ] 可并行：运行 `npm test`。
    - [ ] 可并行：运行 `npm run pack:check`，但必须在清理生成物后执行。
  - 测试方案：
    - `cd services && npm run validate`
    - `cd services && npm test`
    - `cd services && npm run pack:check`
    - `git status --short`
  - 验收标准：
    - 三个 services 门禁均通过。
    - `git status --short` 不出现应忽略生成物。
    - helper 迁移前项目处于可验证状态。
  - 完成总结：
    - 状态：待完成。
    - 变更：待完成。
    - 验证：待完成。
    - 审计与例外：待完成。
    - 下一目标：任务 3.1。

## 3. 抽取低风险 Helper 迁移候选

参考文档：[实施计划 阶段 3](docs/plan/services-sdk-0-6-upgrade-implementation-plan.md#阶段-3抽取低风险-helper-迁移候选)

- [ ] 3.1 生成 helper 迁移候选清单
  - 依赖：任务 2.3。
  - 工作内容：
    - 扫描本地 `grpcCodeFor`、config/secret merge、timeout/TLS、response 读取、JSON parse、脱敏摘要等重复实现。
    - 将候选分为 A 类、B 类、C 类。
    - 明确首批要迁移的 service root；不在清单内的 service 不修改。
    - 将 `mapHttpStatusToCode`、`readResponseJson` 非法 JSON 语义、`google.protobuf.Value` 手写转换默认归为 C 类，除非测试证明可直接替换。
  - 可并行子任务：
    - [ ] 可并行：错误构造和状态码映射扫描。
    - [ ] 可并行：context/config/secret merge 扫描。
    - [ ] 可并行：timeout/TLS/fetch 扫描。
    - [ ] 可并行：response 读取、JSON parse、脱敏摘要扫描。
    - [ ] 可并行：候选 service 测试覆盖审计。
  - 测试方案：
    - `rg -n "const grpcCodeFor|function grpcCodeFor|new GrpcError\\(grpcCodeFor" services/*/src/*.js`
    - `rg -n "AbortController|AbortSignal\\.timeout|makeTimeoutSignal|fetchWithTimeout" services/*/src/*.js`
    - `rg -n "new Agent\\(|import\\('undici'\\)|from 'undici'|from \\"undici\\"" services/*/src/*.js`
    - `rg -n "\\.\\.\\.\\(ctx\\??\\.config \\?\\? \\{\\}\\)|\\.\\.\\.\\(ctx\\??\\.secret \\?\\? \\{\\}\\)" services/*/src/*.js`
  - 验收标准：
    - 候选清单可直接驱动任务 4.1 和任务 5.1。
    - 每个待迁移 service 都有 focused service-local test。
    - C 类保留项已明确，不会在首批迁移中误改。
  - 完成总结：
    - 状态：待完成。
    - 变更：待完成。
    - 验证：待完成。
    - 审计与例外：待完成。
    - 下一目标：任务 4.1。

## 4. 迁移通用 Context 和错误构造 Helper

参考文档：[实施计划 阶段 4](docs/plan/services-sdk-0-6-upgrade-implementation-plan.md#阶段-4迁移通用-context-和错误构造-helper)

- [ ] 4.1 迁移 A 类 context 和错误 helper
  - 依赖：任务 3.1。
  - 工作内容：
    - 对 A 类 service 使用 SDK `grpcCodeFor` 或 `serviceError` 替换本地状态码表，保留既有 message shape、`legacyCode`、`details`、`response` 或 `httpStatus`。
    - 对 A 类 service 使用 `mergeConfigSecret(ctx)` 替换纯 config/secret merge，并保留 `ctx.bindings` 覆盖顺序。
    - 对 metadata helper 候选使用 `getMetadataValue(ctx, key)`，只替换不改变 key 优先级的代码。
    - 不修改 public handler signature。
  - 可并行子任务：
    - [ ] 可并行：按 service root 分片迁移 context merge。
    - [ ] 可并行：按 service root 分片迁移状态码 helper。
    - [ ] 可并行：按 service root 分片补充或调整 focused tests。
  - 测试方案：
    - 对每个修改的 service 运行：
      - `cd services && npm run validate -- --service-dir <service-dir>`
      - `cd services && npm test -- --service-dir <service-dir>`
      - `cd services && npm test -- --coverage --service-dir <service-dir>`
    - 批次完成后运行：
      - `cd services && npm run validate`
      - `cd services && npm test`
  - 验收标准：
    - 被迁移 service 的错误码、message、legacy fields 和测试断言保持不变。
    - `services/scripts/validate-service-package.mjs` 未发现双参数 exported handler。
    - 覆盖率门禁对每个修改 service 通过。
  - 完成总结：
    - 状态：待完成。
    - 变更：待完成。
    - 验证：待完成。
    - 审计与例外：待完成。
    - 下一目标：任务 5.1。

## 5. 迁移 HTTP Timeout、TLS 和 Response 读取 Helper

参考文档：[实施计划 阶段 5](docs/plan/services-sdk-0-6-upgrade-implementation-plan.md#阶段-5迁移-http-timeouttls-和-response-读取-helper)

- [ ] 5.1 迁移 A/B 类 HTTP 底层 helper
  - 依赖：任务 4.1。
  - 工作内容：
    - 对 A 类或明确可控的 B 类 service 使用 `normalizeTimeoutMs`、模块级缓存的 `createTlsDispatcher(true)` 和 `fetchWithTimeout`。
    - 移除被迁移 service 中不再需要的手写 `AbortController`、timeout signal 和重复 `undici.Agent` 创建逻辑。
    - 仅在 body read failure 语义一致时使用 `readResponseText`。
    - 仅在非法 JSON 应映射为 `INTERNAL` 的 service 中使用 `readResponseJson`；否则保留本地 parse wrapper。
    - 仅当 SDK 默认 HTTP status 映射与 service 测试一致时使用 `httpStatusError`；否则保留本地错误映射，只复用 `safeErrorSummary` 或 `redactSensitive`。
  - 可并行子任务：
    - [ ] 可并行：按 service root 分片迁移 timeout/TLS。
    - [ ] 可并行：按 service root 分片迁移 response read 和脱敏摘要。
    - [ ] 可并行：按 service root 分片补充 timeout、TLS skip、network failure、body read failure 或 HTTP status focused tests。
  - 测试方案：
    - 对每个修改的 service 运行：
      - `cd services && npm run validate -- --service-dir <service-dir>`
      - `cd services && npm test -- --service-dir <service-dir>`
      - `cd services && npm test -- --coverage --service-dir <service-dir>`
    - 每个批次完成后运行：
      - `cd services && npm test`
      - `cd services && npm run pack:check`
  - 验收标准：
    - 被迁移 service 不再把 `timeoutMs`、`skipTlsVerify`、`tlsInsecureSkipVerify`、`insecureSkipVerify` 作为伪字段传入原生 `fetch`。
    - 被迁移 service 没有重复 `undici.Agent` 创建逻辑，除非存在特殊 dispatcher 行为并在完成总结中说明。
    - 不改变业务错误码映射和 response field shape。
  - 完成总结：
    - 状态：待完成。
    - 变更：待完成。
    - 验证：待完成。
    - 审计与例外：待完成。
    - 下一目标：任务 6.1。

## 6. 全量 Services 门禁和 Import 验证

参考文档：[实施计划 阶段 6](docs/plan/services-sdk-0-6-upgrade-implementation-plan.md#阶段-6全量-services-门禁和-import-验证)

- [ ] 6.1 运行全量 services 质量门禁
  - 依赖：任务 2.3；如果执行 helper 迁移，还依赖任务 4.1 和任务 5.1。
  - 工作内容：
    - 清理 `services/package-lock.json`、`services/node_modules/`、`*.tgz`、日志、coverage、临时 data dir、`.env`。
    - 运行全量 services validate/test/pack check。
    - 确认 `rg '"@chaitin-ai/octobus-sdk": "\\^0\\.5\\.0"' services` 无结果。
  - 可并行子任务：
    - [ ] 可并行：生成物污染清理和审计。
    - [ ] 可并行：全量 validate。
    - [ ] 可并行：全量 test。
    - [ ] 可并行：pack check。
  - 测试方案：
    - `cd services && npm run validate`
    - `cd services && npm test`
    - `cd services && npm run pack:check`
    - `rg '"@chaitin-ai/octobus-sdk": "\\^0\\.5\\.0"' services`
    - `git status --short`
  - 验收标准：
    - services package 命名、结构、测试和 pack dry-run 均通过。
    - 没有新增 service root、proto、schema、bin 或 dispatcher mapping 变化。
    - 没有应忽略生成物出现在待提交变更中。
  - 完成总结：
    - 状态：待完成。
    - 变更：待完成。
    - 验证：待完成。
    - 审计与例外：待完成。
    - 下一目标：任务 6.2。

- [ ] 6.2 运行 recursive import 验证
  - 依赖：任务 6.1。
  - 工作内容：
    - 构建 `bin/octobus`。
    - 使用 `services/scripts/import-check-all.mjs` 递归导入 services distribution，验证 service ID、ServiceRoot 和 NodeEntry。
  - 可并行子任务：
    - [ ] 可并行：`task build` 构建验证。
    - [ ] 可并行：import check 失败日志审计。
  - 测试方案：
    - `task build`
    - `cd services && npm run import:check -- --octobus ../bin/octobus`
  - 验收标准：
    - `bin/octobus` 构建成功且静态链接检查通过。
    - recursive import check 对 50 个 service 通过。
  - 完成总结：
    - 状态：待完成。
    - 变更：待完成。
    - 验证：待完成。
    - 审计与例外：待完成。
    - 下一目标：任务 6.3。

- [ ] 6.3 条件运行全量 service coverage
  - 依赖：任务 6.1；若 helper 迁移覆盖大量 service 或更改共享模式，则必须执行。
  - 工作内容：
    - 判断 helper 迁移范围是否达到“覆盖大量 service 或更改共享模式”条件。
    - 条件满足时运行全量 `coverage:all`。
    - 条件不满足时，在完成总结中记录未运行原因和已运行的 focused coverage 证据。
  - 可并行子任务：
    - [ ] 可并行：coverage 运行。
    - [ ] 可并行：coverage 失败 service 汇总。
  - 测试方案：
    - 条件满足时：`cd services && npm run coverage:all`
  - 验收标准：
    - 条件满足时，50 个 service coverage 检查全部通过。
    - 条件不满足时，有明确审计说明。
  - 完成总结：
    - 状态：待完成。
    - 变更：待完成。
    - 验证：待完成。
    - 审计与例外：待完成。
    - 下一目标：任务 7.1。

## 7. 仓库级回归和文档收束

参考文档：[实施计划 阶段 7](docs/plan/services-sdk-0-6-upgrade-implementation-plan.md#阶段-7仓库级回归和文档收束)

- [ ] 7.1 复查文档和最终污染状态
  - 依赖：任务 6.1、任务 6.2；如果执行任务 6.3，也依赖任务 6.3。
  - 工作内容：
    - 复查 spec、plan、`PROGRESS.md`、被更新设计文档和被影响 service README。
    - 确认文档中的命令与实际 `services/package.json` scripts、`Taskfile.yml` 和 CI 一致。
    - 做最终污染检查，确认未提交 forbidden/generated artifacts。
  - 可并行子任务：
    - [ ] 可并行：文档链接和命令审计。
    - [ ] 可并行：git status 和 untracked 文件审计。
  - 测试方案：
    - `git status --short`
    - `git ls-files --others --exclude-standard`
  - 验收标准：
    - 文档和实际交付状态一致。
    - 没有 `services/package-lock.json`、`node_modules/`、pack artifact、日志、coverage、`.env` 或 secret 进入待提交变更。
  - 完成总结：
    - 状态：待完成。
    - 变更：待完成。
    - 验证：待完成。
    - 审计与例外：待完成。
    - 下一目标：任务 7.2。

- [ ] 7.2 运行仓库级 harness 门禁
  - 依赖：任务 7.1。
  - 工作内容：
    - 运行仓库级 lint/test/build。
    - 如果改动影响 package import、routing protocol、supervision、CLI 或 daemon startup，追加运行 e2e。
    - 如果 Go/e2e 失败且与本变更无关，记录证据，不混入无关修复；如果相关，修复后重跑。
  - 可并行子任务：
    - [ ] 可并行：`task lint`。
    - [ ] 可并行：`task test`。
    - [ ] 可并行：`task build`。
    - [ ] 可并行：条件 e2e。
  - 测试方案：
    - `task lint`
    - `task test`
    - `task build`
    - 条件触发时：`go test ./tests/e2e -count=1`
  - 验收标准：
    - 仓库级 lint/test/build 通过。
    - 条件触发时 e2e 通过，或完成总结中有明确未运行原因。
    - PR 摘要可列出 SDK dependency 升级、helper 迁移服务列表和实际运行门禁。
  - 完成总结：
    - 状态：待完成。
    - 变更：待完成。
    - 验证：待完成。
    - 审计与例外：待完成。
    - 下一目标：无。

## 首版不做的事项

- 不升级 `examples/*` 的 SDK dependency。
- 不发布或改名 `@chaitin-ai/octobus-tentacles`。
- 不修改 SDK 源码、SDK 发布流程或 SDK version。
- 不迁移 services validator/dispatcher 到 SDK multi-service CLI。
- 不删除 service root 的 `undici` 直接依赖，除非对应 service 源码已完成迁移并通过 focused 门禁。
- 不批量替换会改变语义的 HTTP status 映射、非法 JSON 映射或 protobuf `google.protobuf.Value` 手写转换。
- 不改变 proto、schema、service name、bin、handler key、runtime mode 或上游业务字段。

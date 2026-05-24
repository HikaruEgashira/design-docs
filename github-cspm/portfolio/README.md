# GitHub CSPM — portfolio design doc

> design doc / 統合 thesis。読者は自分と将来の協力者。これは「個別に作ってきた GitHub
> セキュリティ OSS 群が、実は一つの製品圏(GitHub CSPM)を構成している」ことを自問検証し、
> 統合アーキテクチャと残課題を一枚に束ねる思考アーティファクトである。セールス資料ではない。
>
> 各節末の `[...]` は PMF harness の次元タグ(why-now / wedge / TTFV / trust / durability / moat)。
> 製品ごとの記述は README / コードから確認した `事実:`。それらを束ねる「GitHub CSPM」という
> framing は `推測:`(私の統合解釈)であり、本書はその spec を提示するもの。

---

## 0. TL;DR — 3-sentence thesis

**GitHub は組織の de-facto cloud control plane になった** — IAM(branch protection / permissions)、compute(Actions runner)、artifact registry(releases / packages)、audit log を一通り持つ。**だが既存の CSPM/CNAPP(Wiz, Prisma, Orca…)は IaaS(AWS/GCP/Azure)を見ており、GitHub 自身の posture は死角**である。**私が別々に作ってきた OSS 群は、この死角を posture / pipeline / detection の3層で既に埋めており、本書はそれらを一つの GitHub CSPM reference architecture として束ねる。**

---

## 1. なぜ "GitHub CSPM" か `[why-now / wedge]`

### GitHub = cloud という見立て

CSPM(Cloud Security Posture Management)が IaaS に対して行うことを、構成要素ごとに GitHub に写像できる:

| CSPM が cloud で見るもの | GitHub における対応物 |
|---|---|
| 資産インベントリ(VM/bucket/role) | repo / org / team / secret / OIDC role / self-hosted runner |
| IAM 設定の最小権限 | branch protection / `permissions:` / collaborator / app installation |
| compute の実行時挙動 | GitHub Actions runner 上のプロセス挙動 |
| supply chain(image/pkg) | releases / dependencies / reusable workflows / marketplace actions |
| 監査ログの異常検知 | org/enterprise audit log / workflow run イベント |
| compliance(CIS/PCI) | SLSA L1-L4 / SOC2 CC7-CC8 / ISMAP |

`推測:` 既存 CSPM が GitHub を一級の対象にしない理由は、彼らの収集 primitive が cloud provider API(CloudTrail, Config)に最適化されており、GitHub の内部 posture(PR ceremony, workflow taint, OIDC trust policy)を評価する control 体系を持たないこと。根拠 — Wiz/Prisma の GitHub 連携は「secret scanning の結果を取り込む」程度に留まり、branch protection の stale-review や workflow の expression injection を verdict にしない。

### why-now

`事実:` 2024-2025 に GitHub 自身を狙う supply-chain 攻撃が一級化した — `tj-actions/changed-files`(2025/03)、`xz-utils`、`event-stream`、`ua-parser-js`。攻撃面は「コードの脆弱性」から「SDLC/CI-CD パイプラインの構成と信頼」へ移った。**posture(設定が安全か)・pipeline(CI/CD が汚染されていないか)・detection(今まさに改竄されていないか)を一つの面として管理する需要**が立ち上がっている。これが GitHub CSPM の why-now。

→ 関連 thesis: [`libverify/agent-era-verification`](../../libverify/agent-era-verification/README.md)(検証単位が PR → agent execution へ移っても verdict 需要は残る、という durability 論)。GitHub CSPM はその posture 層の宿主。

---

## 2. CSPM reference architecture — capability map `[wedge]`

CNAPP の標準能力を縦軸に、私の製品をセルに割り当てる。**空白セルが gap(§5)。**

| CNAPP capability | 担当製品 | layer |
|---|---|---|
| 設定 posture verify(repo/PR/release) | **libverify** / **gh-verify** / **metsuke** | 1 |
| compliance mapping(SLSA/SOC2/ISMAP) | **libverify**(profiles)/ **metsuke**(presets) | 1 |
| IaC posture(OIDC/IAM least-privilege) | **gata** | 1 |
| CI/CD pipeline SAST(workflow injection 等) | **sisakulint** family | 2 |
| 依存・install-time hardening | **pmsec** | 2 |
| benchmark / 検証 harness | **github-actions-goat** / **workflow-probe** | 2 |
| audit log 異常検知(CDR) | **gh-intel** | 3 |
| supply-chain 改竄検知 | **sisakudyscan** | 3 |
| 外部 OSS の CI/CD posture スキャン | **sisakuintel-worker** | 3 |
| runtime/honeypot telemetry | **sisakuintel-pot** | 3 |
| 動的(eBPF)検知 R&D | **agent-idea** | 3(idea) |
| auto-remediation | **sisakulint**(autofix)/ **pmsec**(--disable) | 2 |

---

## 3. 三層の製品群

製品ごとの記述は README / コードから確認した `事実:`。owner を明記(personal `HikaruEgashira` / org `plenoai` / org `sisaku-security`)。

### Layer 1 — Posture & Compliance(設定を verify する)

| 製品 | owner | 一行 positioning | 入力 → 出力 | 成熟度 |
|---|---|---|---|---|
| **libverify** | HikaruEgashira | platform-agnostic SDLC 検証エンジン。"libghostty for SDLC verification"。control が evidence → verdict(Satisfied/Violated/Indeterminate/N-A)を返し、OPA profile が gate 決定。判定述語は Creusot で形式証明 | evidence bundle → verdict(SARIF/JSON) | 0.x active |
| **gh-verify** | HikaruEgashira | GitHub SDLC health checker。`gh` CLI extension。libverify を GitHub に bind。stale-review / commit signing / PR quality / dep provenance を検査 | GitHub API → verdict / SARIF | 0.13、semver |
| **metsuke** | plenoai | GitHub SDLC health check **Platform**。libverify を載せた hosted SaaS = Remote MCP Server + GitHub App。webhook で PR/release を継続検証し Check Run を投稿。OAuth 2.1、9 profile presets、self-host(GHES)可 | webhook/MCP → Check Run / dashboard / audit log | 0.2.0 beta、Fly.io(nrt) |
| **gata** | sisaku-security | GitHub Actions Terraform Auditor。Actions OIDC role の IAM policy を静的監査し least-privilege を強制。`dangerous-action`/`disallowed-action`/`missing-oidc-module` | Terraform HCL → linter findings + exit code | prod-grade |

**構造:** libverify(engine)= 中核。gh-verify(local CLI wedge)と metsuke(hosted platform)は同じエンジンの thin shell。これは「鋭い楔が再利用可能な深い engine を front する」libghostty パターンそのもの。gata は IaC 側から同じ posture 思想(least-privilege gate)を担う独立 linter。

### Layer 2 — Pipeline security(CI/CD を SAST + hardening)

| 製品 | owner | 一行 positioning | 備考 |
|---|---|---|---|
| **sisakulint** | sisaku-security | GitHub Actions の semantic SAST/linter(50+ rules、27+ autofix、SARIF)。AST + expression parser + shell taint(mvdan.cc/sh)で cross-step/cross-file の taint 伝播を追う。regex ではない。OWASP CI/CD Top 10 全網羅、AI-action security も | flagship。Go、Apache 2.0、BlackHat Asia 2025 Arsenal、SecHack365 2023 起源 |
| **sisakulint-action** | sisaku-security | 公式 GitHub Action。inline PR annotation + Code Scanning への SARIF upload + org-wide enforcement(ruleset) | 配布層(PR-native) |
| **sisakulint-api** | sisaku-security | remote scan の managed API(AWS Lambda + API Gateway) | 配布層(programmatic) |
| **sisakulint-agent** | sisaku-security | "run remotely"。PR event で自動スキャンし `/autofix` slash command を提供する GitHub App(Supabase Functions) | 配布層(webhook) |
| **sisakulint-for-ai** | sisaku-security | Claude Code 等の agent skill。install→review→autofix を誘導 | 配布層(agent wedge) |
| **workflow-probe** | sisaku-security | runner 実機挙動が linter の主張と一致するか検証する harness(spec QA) | 検証 harness(製品でない) |
| **github-actions-goat** | sisaku-security | 意図的に脆弱な Actions 環境 = SAST ベンチ/教育(StepSecurity fork) | benchmark/教育 |
| **pmsec** | HikaruEgashira | npm/pnpm/yarn/bun/cargo/mise/uv の **install-time** zero-config hardening。`npx pmsec`/`uvx pmsec`、`--check`/`--disable`/`--doctor` | host/dev-env scope。GitHub 外も含む supply-chain 入口の hardening |

**構造:** sisakulint(offline static engine)= 中核。action / api / agent / for-ai は配布 form factor(PR / API / webhook / agent)。これも engine + 複数 shell パターン。pmsec は CI ではなく開発機の依存導入時を守る別 vector(supply-chain の最上流)。

### Layer 3 — Detection & Response(runtime / audit の脅威検知 = CDR)

| 製品 | owner | 一行 positioning | 入力 → 出力 | 成熟度 |
|---|---|---|---|---|
| **gh-intel** | HikaruEgashira(mirror: sisaku-security) | Agentic SOC / threat intelligence。org polling と enterprise streaming(S3/GCS)で audit log を取り込み rule で finding 化、agent(Claude/Codex/Pi)が investigate。**PAT auth を意図的に拒否** | audit log → Finding → agent 調査 | prod MVP、DuckDB 圧縮 |
| **sisakudyscan** | sisaku-security | Threat Detection for GitHub。supply-chain 改竄シグナルを検知(17 rules / 7 critical)。`pr-target-abuse`/`release-actor-anomaly`/`commit-date-gap`/`null-committer-email` 等 | GitHub API → JSON findings | prod、Postgres |
| **sisakuintel-worker** | sisaku-security | CICD Threat Intelligence System。OSS Insight で trending repo を発見→sisakulint API で workflow 監査→対象 repo に Issue + Discord 通知。Issue が triage state machine | trending → scan → Issue | prod、Deno、daily cron |
| **sisakuintel-pot** | sisaku-security | honeypot for GitHub。self-hosted runner で攻撃挙動を捕捉(Vector→S3)、audit log(gh-intel→S3)と合流し Athena/Parquet で分析 | runner syslog + audit → Parquet | prod infra、実トラフィック捕捉 |
| **agent-idea** | sisaku-security | 動的解析の R&D。eBPF hook / API monitor / attack-chain correlation / 60+ bug bounty report 索引、MITRE ATT&CK・OWASP CI/CD マッピング | (research only) | idea stage |

**構造:** detection 層は posture 層と逆向き — `signal → finding → investigate`。gh-intel(audit log の SOC)、sisakudyscan(改竄の supply-chain detection)、sisakuintel-*(外部 OSS スキャン + honeypot による intel 生成)が並ぶ。agent-idea が次世代の runtime(eBPF)検知の種。

---

## 4. 統合アーキテクチャ — なぜ一つの CSPM になるか `[moat]`

`推測:` 3層は別々に育ったが、共通の骨格を持つ。

```
                       ┌──────────────── GitHub (= cloud control plane) ────────────────┐
                       │  repo/org settings │ PR/release │ Actions runner │ audit log    │
                       └──────┬─────────────────┬──────────────┬──────────────┬──────────┘
            posture           │                 │              │              │  detection
        (evidence→verdict)    ▼                 ▼              ▼              ▼ (signal→finding)
   ┌──────────────────────────────────┐   ┌──────────┐   ┌──────────┐  ┌───────────────────────┐
   │ libverify (engine, 形式証明)       │   │sisakulint │   │  gata    │  │ gh-intel / sisakudyscan│
   │   ├─ gh-verify  (CLI wedge)        │   │ (CI SAST) │   │(IaC/OIDC)│  │ sisakuintel-* (CDR)    │
   │   └─ metsuke    (hosted platform)  │   │ +action/  │   └──────────┘  │ + honeypot intel       │
   │       ▲ aggregator                 │   │  api/agent│                 └───────────┬───────────┘
   └───────┼────────────────────────────┘   └─────┬─────┘                             │
           │            verdict / finding を共通 contract(SARIF / OTel)で集約          │
           └────────────────────────────── metsuke = 集約点 ◄───────────────────────────┘
                                  │
                       data flywheel: 使用 → telemetry(verdict+override / finding+triage)
                                     → FP/TP label → 自動 healing → 精度↑ → 信頼↑ → 使用↑
```

統合の3本の糸:

1. `事実:` **engine + thin shell が両 posture 製品で共通**。libverify→{gh-verify, metsuke}、sisakulint→{action, api, agent, for-ai}。どちらも「深い engine を鋭い楔が front する」同型。
2. `推測:` **metsuke が aggregator になれる**。既に libverify の hosted 層であり、MCP/webhook で継続データを持つ。ここに sisakulint findings(SARIF)と gh-intel/sisakudyscan の detection を流し込めば、posture+pipeline+detection が一つの dashboard に合流する。
3. `推測:` **moat は data flywheel**。engine は clone され得るが、telemetry で healing された精度(verdict の FP/TP、finding の triage 結果)は clone できない。posture の override 信号と detection の triage 信号は、どちらも同じ「正解ラベル」を engine に還流する。これが [`libverify/agent-era-verification`](../../libverify/agent-era-verification/README.md) §5-7 の OTel contract と整合。

---

## 5. 重複・摩擦・gap — 正直に `[trust / durability]`

統合を阻む技術負債。Severity 付き。

- `High` **出力 contract が不統一。** posture 系(libverify/gh-verify/metsuke)は SARIF/JSON、sisakulint も SARIF を出すが、detection 系(gh-intel/sisakudyscan)は custom JSON。§4 の「共通 contract で集約」は現状未実現。**統合の前提なので最優先。** → SARIF + OTel を全製品の egress 標準に揃える。
- `High` **detection が realtime でない。** gh-intel(polling)/ sisakudyscan(batch)/ sisakuintel-worker(daily cron)はいずれも非 webhook。CDR を名乗るなら near-real-time ingestion が要る。agent-idea の eBPF 構想はここを埋める種だが idea stage。
- `Medium` **gh-intel が2 owner に存在**(HikaruEgashira と sisaku-security)。どちらが canonical か不明瞭。統合時に source of truth を1つに。
- `Medium` **posture と detection が別 datastore**(metsuke=SQLite / sisakudyscan=Postgres / sisakuintel=Athena-Parquet / gh-intel=DuckDB)。aggregator(metsuke)に集約する経路が無い。
- `Medium` **naming/branding が4系統に分裂**(`pleno-*` / `sisaku*` / `gh-*` / `lib*`)。"GitHub CSPM" という統一傘がプロダクト名として存在しない。採用者から見て「これらが一つの製品圏」と認識されない = wedge が分散。
- `Low` **github-actions-goat / workflow-probe は製品でなく harness/benchmark**。capability map に載るが収益・採用の対象外。混同しない。
- `Low` **pmsec / parsentry は GitHub 外スコープを含む**。pmsec は開発機、parsentry はコード SAST。CSPM 中核ではなく隣接(§範囲外)。

---

## 6. open decisions

| # | decision | 状態 |
|---|---|---|
| 1 | "GitHub CSPM" を製品傘ブランドにするか、内部 framing に留めるか | 未定。ブランド化すれば wedge 統合、しなければ各 OSS は独立のまま |
| 2 | 集約点を metsuke にするか、別の CNAPP dashboard を新設するか | 未定。metsuke は posture aggregator だが detection ingestion を持たない |
| 3 | 全製品の egress を SARIF + OTel に統一する範囲と順序 | 未定(§5 High)。posture 系は近い、detection 系の custom JSON が遠い |
| 4 | gh-intel の canonical owner(HikaruEgashira vs sisaku-security) | 未定(§5 Medium) |
| 5 | realtime ingestion(webhook / streaming)を detection 層に入れるか | 未定。agent-idea の eBPF/API monitor 構想を製品化するか |
| 6 | compliance presets を全層横断で1つの control catalog に統合するか | 未定。libverify(34 controls)/ metsuke(28)/ sisakulint(50+)/ sisakudyscan(17)/ gata が別々の rule 体系 |

推奨 follow-up: §5 High の2点(SARIF/OTel 統一 + realtime)が「3層が一つの CSPM になる」かを決める律速。まず decision #3 の contract 統一から着手し、metsuke を集約点として #2 を検証する。

---

## 範囲外の観察(GitHub CSPM 中核の外)

`事実:` 同じセキュリティ portfolio だが GitHub posture 管理ではない隣接プロダクト。混同しないため分離:

- `Low` **parsentry**(HikaruEgashira)— AI agent 向け security scan orchestrator。コード SAST であり GitHub 設定 posture ではない。
- `Low` **cve-hunter / phishing-hunter**(HikaruEgashira)— CVE 探索 / ブランド偽装フィッシング検知。GitHub 面ではない。
- `Low` **pleno-dlp / pleno-anonymize / pleno-audit / pleno-cockpit / pleno-live**(plenoai)— DLP / PII / browser zero-trust / runtime enforcement / 文字起こし。plenoai の別プロダクト群で、本書の CSPM 軸とは別。ただし pleno-cockpit(runtime enforcement)は [`libverify/agent-era-verification`](../../libverify/agent-era-verification/README.md) §6 で「enforcement vs verification」の対として既出。

---

## References

- 関連 thesis: [`libverify/agent-era-verification`](../../libverify/agent-era-verification/README.md)
- Layer 1: [libverify](https://github.com/HikaruEgashira/libverify) · [gh-verify](https://github.com/HikaruEgashira/gh-verify) · [metsuke](https://github.com/plenoai/metsuke) · [gata](https://github.com/sisaku-security/gata)
- Layer 2: [sisakulint](https://github.com/sisaku-security/sisakulint) · [sisakulint-action](https://github.com/sisaku-security/sisakulint-action) · [pmsec](https://github.com/HikaruEgashira/pmsec) · [github-actions-goat](https://github.com/sisaku-security/github-actions-goat)
- Layer 3: [gh-intel](https://github.com/HikaruEgashira/gh-intel) · [sisakudyscan](https://github.com/sisaku-security/sisakudyscan) · [sisakuintel-worker](https://github.com/sisaku-security/sisakuintel-worker) · [sisakuintel-pot](https://github.com/sisaku-security/sisakuintel-pot)

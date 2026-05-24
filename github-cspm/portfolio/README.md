# GitHub CSPM — portfolio design doc

> design doc / 統合 thesis。読者は自分と将来の協力者。これは「個別に作ってきた GitHub
> セキュリティ OSS が、実は一つの製品圏(GitHub CSPM)を構成している」ことを自問検証し、
> 統合アーキテクチャと残課題を一枚に束ねる思考アーティファクトである。セールス資料ではない。
>
> **scope:** 本書は **HikaruEgashira 本人が主筆した製品**のみを「自作」として扱う(git authorship で確認)。
> 中核は **GitHub threat intelligence / detection(intel 系)**。posture 系がそれを支える第二の柱。
> 他者が主筆する OSS(sisakulint 本体など)は upstream/連携として参照し、自作とは区別する。
>
> 各節末の `[...]` は PMF harness の次元タグ(why-now / wedge / TTFV / trust / durability / moat)。

---

## 0. TL;DR — 3-sentence thesis

**GitHub は組織の de-facto cloud control plane になった** — IAM(branch protection / permissions)、compute(Actions runner)、artifact registry(releases / packages)、audit log を一通り持つ。**だが既存の CSPM/CNAPP(Wiz, Prisma, Orca…)は IaaS(AWS/GCP/Azure)を見ており、GitHub 自身の posture と脅威検知は死角**である。**私の OSS 群はこの死角を埋めており、中核は GitHub の audit log / runner / ecosystem から脅威 intel を生成・検知する detection 層(intel 系)、それを posture 検証層(libverify/metsuke)が支える — 両者を一つの GitHub CSPM として束ねる。**

---

## 1. なぜ "GitHub CSPM" か `[why-now / wedge]`

### GitHub = cloud という見立て

CSPM/CNAPP が IaaS に対して行うことを、構成要素ごとに GitHub に写像できる:

| CNAPP が cloud で見るもの | GitHub における対応物 | この portfolio の担当 |
|---|---|---|
| 監査ログの異常検知(CDR) | org/enterprise audit log | **gh-intel**(中核) |
| 脅威 intel 生成(honeypot / ecosystem) | runner 攻撃挙動 / trending OSS の CI 構成 | **sisakuintel-pot / -worker**(中核) |
| 設定 posture / 最小権限(CSPM) | branch protection / `permissions:` / PR ceremony | **libverify / gh-verify / metsuke** |
| compliance(CIS/PCI 相当) | SLSA L1-L4 / SOC2 CC7-CC8 / ISMAP | libverify(profiles)/ metsuke(presets) |
| supply chain / install hardening | dependencies / package manager | **pmsec** |
| CI/CD pipeline SAST | workflow injection / taint | *upstream: sisakulint(§3.4)を連携消費* |

`推測:` 既存 CSPM が GitHub を一級の対象にしない理由は、収集 primitive が cloud provider API(CloudTrail, Config)に最適化されており、GitHub の内部 signal(audit log の意味、workflow taint、runner 上の攻撃挙動)を intel/verdict にする体系を持たないこと。根拠 — Wiz/Prisma の GitHub 連携は secret scanning 結果の取り込み程度に留まる。

### why-now

`事実:` 2024-2025 に GitHub 自身を狙う supply-chain 攻撃が一級化した — `tj-actions/changed-files`(2025/03)、`xz-utils`、`event-stream`、`ua-parser-js`。攻撃面は「コードの脆弱性」から「SDLC/CI-CD パイプラインの構成と信頼」へ移った。**今まさに改竄されていないか(detection)を audit log と ecosystem から捉える需要**が立ち上がっている。これが intel 系の why-now。

→ 関連 thesis: [`libverify/agent-era-verification`](../../libverify/agent-era-verification/README.md)(検証単位が PR → agent execution へ移っても verdict 需要は残る、という posture 層の durability 論)。

---

## 2. 中核 — GitHub threat intelligence / detection(intel 系)`[wedge / moat]`

`事実:` 本人が主筆(git authorship 確認済み)。intel 系3製品が一つの「脅威 intel 生成 → 検知 → 調査」ループを成す。

| 製品 | owner | 一行 positioning | 入力 → 出力 | 成熟度 |
|---|---|---|---|---|
| **gh-intel** | HikaruEgashira(sisaku-security に mirror) | Agentic SOC / threat intelligence。org polling と enterprise streaming(S3/GCS)で audit log を取り込み rule で finding 化、agent(Claude/Codex/Pi)が investigate。**PAT auth を意図的に拒否**(GitHub App / streaming のみ) | audit log → Finding → agent 調査 | prod MVP、DuckDB 圧縮 |
| **sisakuintel-worker** | HikaruEgashira(sisaku-security) | CICD Threat Intelligence System。OSS Insight で trending repo を発見 → workflow を SAST 監査(後述 sisakulint を連携) → 対象 repo に Issue + Discord 通知。Issue が triage state machine | trending OSS → scan → Issue/Discord | prod、Deno、daily cron |
| **sisakuintel-pot** | HikaruEgashira(sisaku-security) | honeypot for GitHub。self-hosted runner で攻撃挙動を捕捉(Vector→S3)、gh-intel の audit log(→S3)と合流し Athena/Parquet で分析。実トラフィック捕捉済み | runner syslog + audit log → Parquet | prod infra |

**intel ループ:**

```
sisakuintel-pot   ──攻撃 TTP 捕捉──┐
(honeypot runner)                 │
sisakuintel-worker ──ecosystem──┐ ├──► 脅威 intel ──► detection rules ──► gh-intel
(trending OSS scan)             │ │                                      (自組織 audit log で検知)
                                ▼ ▼                                            │
                          Athena/Parquet(intel store)                  agent が investigate
```

- **pot** = adversary の TTP を一次取得(intel の供給源)。
- **worker** = ecosystem(trending OSS の CI 構成)を外向きにスキャンし intel 化。
- **gh-intel** = 自組織の audit log に対し detection を回し、agent が調査。

`推測:` この3つは「intel を生成する側(pot/worker)」と「intel を消費して自組織を守る側(gh-intel)」の閉ループで、これが本 portfolio の moat 候補 — honeypot 由来の一次 intel は clone しにくい。根拠 — pot は実 runner 上の実トラフィックに依存し、データ自体が参入障壁。

---

## 3. 支える柱・連携

### 3.1 Posture & Compliance(設定を verify する第二の柱)`[durability]`

`事実:` 本人が主筆(metsuke は 211 commits で本人 dominant)。

| 製品 | owner | 一行 positioning | 入力 → 出力 |
|---|---|---|---|
| **libverify** | HikaruEgashira | platform-agnostic SDLC 検証エンジン。"libghostty for SDLC verification"。control が evidence → verdict(Satisfied/Violated/Indeterminate/N-A)、OPA profile が gate 決定。判定述語を Creusot で形式証明 | evidence → verdict(SARIF/JSON) |
| **gh-verify** | HikaruEgashira | GitHub SDLC health checker。`gh` CLI extension で libverify を GitHub に bind。stale-review / commit signing / PR quality / dep provenance | GitHub API → verdict/SARIF |
| **metsuke** | plenoai | GitHub SDLC health check **Platform**。libverify を載せた hosted SaaS = Remote MCP Server + GitHub App。webhook で PR/release を継続検証し Check Run を投稿。OAuth 2.1、9 profile presets、self-host(GHES)可 | webhook/MCP → Check Run/dashboard |

**構造:** libverify(engine)= 中核。gh-verify(local CLI)と metsuke(hosted platform)は同じエンジンの thin shell — 「鋭い楔が再利用可能な深い engine を front する」libghostty パターン。intel 系が detection(事後)を担うのに対し、posture 系は prevention/verification(事前〜継続)を担う。

### 3.2 Supply-chain hardening

| 製品 | owner | 一行 positioning |
|---|---|---|
| **pmsec** | HikaruEgashira | npm/pnpm/yarn/bun/cargo/mise/uv の **install-time** zero-config hardening。`npx pmsec` / `uvx pmsec`、`--check`/`--disable`/`--doctor`。開発機 = supply-chain の最上流を守る |

### 3.3 upstream / 連携(他者の中核 + 自作の distribution 層)

`事実:` **sisakulint 本体は ultra-supara(atsushi sada)氏が主筆**(本人ではない)。本人は **その上の distribution/連携層**(下記)を書いた。intel 系 §2 がこれを CI/CD SAST の signal source として消費する。**自作プロダクトの headline ではなく、連携 context として記述する。**

| repo | 主筆 | 役割 | 本 portfolio との関係 |
|---|---|---|---|
| **sisakulint** | ultra-supara(他者) | GitHub Actions の semantic SAST(50+ rules、autofix、SARIF)。AST + expression + shell taint | upstream engine。sisakuintel-worker が分析エンジンとして利用 |
| sisakulint-api | HikaruEgashira | remote scan の managed API(Lambda) | 本人作の distribution。worker が叩く |
| sisakulint-agent | HikaruEgashira | "run remotely"。PR event で自動スキャン + `/autofix`(GitHub App) | 本人作の distribution |
| sisakulint-action / -for-ai | HikaruEgashira | 公式 Action / Claude Code skill | 本人作の distribution |

`推測:` ここは「他者の中核を、本人が API/agent/action として productize し、自分の intel pipeline に繋いだ」構造。中核の所有権は upstream にあるため、CSPM portfolio 上は **自社 intel が外部 SAST を取り込む連携点**として位置づけるのが正直。

---

## 4. 統合アーキテクチャ — なぜ一つの CSPM になるか `[moat]`

```
              ┌──────────────── GitHub (= cloud control plane) ────────────────┐
              │  repo/org settings │ PR/release │ Actions runner │ audit log     │
              └──────┬───────────────────┬───────────────┬────────────┬─────────┘
   posture           │                   │               │            │  detection / intel(中核)
(evidence→verdict)   ▼                   ▼               ▼            ▼ (signal→intel→finding)
   ┌─────────────────────────────────┐  ┌────────────┐  ┌──────────────────────────────────┐
   │ libverify (engine, 形式証明)      │  │ sisakulint │  │ sisakuintel-pot (honeypot TTP)     │
   │   ├─ gh-verify (CLI)             │  │ (upstream, │  │ sisakuintel-worker (ecosystem scan)│
   │   └─ metsuke  (hosted platform)  │  │  他者)─────┼─►│ gh-intel (audit log SOC + agent)   │
   │        ▲ aggregator 候補          │  └────────────┘  └─────────────┬────────────────────┘
   └────────┼─────────────────────────┘                                │
            │     verdict / finding を共通 contract(SARIF/OTel)で集約    │
            └──────────────── metsuke = 集約点候補 ◄─────────────────────┘
                         │
              data flywheel: 使用 → telemetry(verdict+override / finding+triage)
                            → FP/TP label → 自動 healing → 精度↑ → 信頼↑ → 使用↑
```

統合の糸:

1. `事実:` **intel 系がループを閉じている**(§2)。pot/worker が intel を供給し、gh-intel が自組織で検知する。
2. `推測:` **posture 系の engine + thin shell**(libverify→{gh-verify, metsuke})が同型の moat 構造。metsuke は既に hosted aggregator なので、intel findings と posture verdict の集約点候補。
3. `推測:` **moat = データ**。posture 側は telemetry-healed 精度(verdict の FP/TP)、intel 側は honeypot 由来の一次 TTP。どちらも clone しにくいデータ資産。[`libverify/agent-era-verification`](../../libverify/agent-era-verification/README.md) §5-7 の OTel contract と整合。

---

## 5. 重複・摩擦・gap — 正直に `[trust / durability]`

Severity 付き。

- `High` **出力 contract が不統一。** posture 系(libverify/gh-verify/metsuke)は SARIF/JSON、intel 系(gh-intel/sisakuintel-*)は custom JSON / Parquet。§4 の「共通 contract で集約」は未実現。統合の前提なので最優先 → SARIF + OTel を egress 標準に揃える。
- `High` **intel 系が realtime でない。** gh-intel(polling)/ sisakuintel-worker(daily cron)はいずれも非 webhook。CDR を名乗るなら near-real-time ingestion が要る。
- `High` **CI/CD SAST が他者依存。** §2 worker の分析エンジン(sisakulint)は upstream(ultra-supara 氏)所有。portfolio の中核機能の一つが外部依存 — fork/置換可能性を設計に持つか、依存を明示するか要判断。
- `Medium` **posture と intel が別 datastore**(metsuke=SQLite / gh-intel=DuckDB / sisakuintel=Athena-Parquet)。集約点(metsuke)への経路が未敷設。
- `Medium` **gh-intel が2 owner に存在**(HikaruEgashira / sisaku-security mirror)。canonical を1つに。
- `Medium` **naming/branding が分裂**(`gh-*` / `sisakuintel-*` / `pleno-*` / `lib*`)。"GitHub CSPM" という統一傘がプロダクト名として不在 = 採用者に「一つの製品圏」と認識されない。

---

## 6. open decisions

| # | decision | 状態 |
|---|---|---|
| 1 | "GitHub CSPM" を製品傘ブランドにするか、内部 framing に留めるか | 未定 |
| 2 | 集約点を metsuke にするか、intel 専用 store(Athena)を中心にするか | 未定。posture は metsuke、intel は Parquet で分断中 |
| 3 | 全製品の egress を SARIF + OTel に統一する範囲と順序 | 未定(§5 High) |
| 4 | CI/CD SAST(sisakulint)依存をどう扱うか(連携明示 / 内製化 / 抽象化) | 未定(§5 High) |
| 5 | intel 系に realtime ingestion(webhook/streaming)を入れるか | 未定 |
| 6 | gh-intel の canonical owner | 未定 |

推奨 follow-up: §5 High 3点(contract 統一 / realtime / SAST 依存)が「intel + posture が一つの CSPM になる」かの律速。まず #3 の contract 統一から着手し、metsuke を集約点として #2 を検証する。

---

## 範囲外の観察

`事実:` 同 org / 同ジャンルだが、本書の「本人主筆 × GitHub CSPM」から外れるもの。混同を避けるため分離:

- `Low` **他者主筆のため除外**(git authorship): sisakulint 本体・**gata**(ともに ultra-supara 氏)、**agent-idea / workflow-probe**(on-keyday 氏)、**github-actions-goat**(StepSecurity の fork)。
- `Low` **本人申告により除外:** **sisakudyscan**(本人のものではない)。
- `Low` **本人作だが GitHub posture 外の隣接プロダクト:** parsentry(コード SAST)、cve-hunter / phishing-hunter、pleno-dlp / pleno-anonymize / pleno-audit / pleno-cockpit(plenoai の別ジャンル)。ただし pleno-cockpit(runtime enforcement)は [`libverify/agent-era-verification`](../../libverify/agent-era-verification/README.md) §6 で enforcement vs verification の対として既出。

---

## References

- 関連 thesis: [`libverify/agent-era-verification`](../../libverify/agent-era-verification/README.md)
- 中核(intel): [gh-intel](https://github.com/HikaruEgashira/gh-intel) · [sisakuintel-worker](https://github.com/sisaku-security/sisakuintel-worker) · [sisakuintel-pot](https://github.com/sisaku-security/sisakuintel-pot)
- posture: [libverify](https://github.com/HikaruEgashira/libverify) · [gh-verify](https://github.com/HikaruEgashira/gh-verify) · [metsuke](https://github.com/plenoai/metsuke)
- hardening: [pmsec](https://github.com/HikaruEgashira/pmsec)
- upstream/連携: [sisakulint](https://github.com/sisaku-security/sisakulint)(ultra-supara) · [sisakulint-agent](https://github.com/sisaku-security/sisakulint-agent) · [sisakulint-api](https://github.com/sisaku-security/sisakulint-api)

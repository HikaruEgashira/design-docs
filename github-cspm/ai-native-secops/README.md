# GitHub CSPM — AI-native security operations(構想)

> design doc / thesis。読者は自分と将来の協力者。これは「個別 OSS の羅列」から「一つの賭け」へ
> 昇格させるための思考アーティファクト。grill-me による自問の結論を固定したもの。セールス資料ではない。
>
> 各節末の `[...]` は PMF harness の次元タグ(why-now / wedge / TTFV / trust / durability / moat)。
> `事実(確定):` は grill で本人が確定した設計。`推測:` は私の統合解釈。
> 関連: [`libverify/agent-era-verification`](../../libverify/agent-era-verification/README.md)

---

## 0. TL;DR — 3-sentence thesis

**GitHub セキュリティ運用は「人間が platform を操作する」前提では回らない**(metsuke の死がそれを示した) — 運用は AI agent が実施しなければならない。**AI で運用が完結するには、real-time 検知が "実行起点(trigger)" かつ "制御点(control point)" になり、AI agent がそれを起点に investigate して control(是正)し、posture(宣言的 desired-state)が全 control を可逆にする** — これが自動化のパラドックスを解き、スキル人材のいない組織でも回せる唯一の形。**GitHub CSPM = この AI-complete な「検知 → 調査 → 制御」ループであり、本命は real-time trigger をシステム側に提供できる唯一の form factor = gh-intel。**

---

## 1. why-now — metsuke の死から学んだこと `[why-now]`

`事実(確定):` gh-verify / metsuke は **死にゆく prototype**。価値創出が難しく、世に出さず消える公算が大きい。診断:

- **metsuke が失敗した真因 = platform 化したが「人間が操作する」前提だった。** 人間運用は難しすぎた。
- **CLI(gh-verify / dyscan)は AI が運用実行できる**が、①real-time 性がなく、②「システムに組み込まれる」のでなく「**agent に組み込まれる**」ため、**実行起点も制御点も置けない**。AI が「呼べば動く」だけでは運用は完結しない。

→ 結論: **AI で完結する運用 = real-time 検知 → それを起点に AI agent が動いて control する。** trigger と control point は **システム側**に無ければならない。これが why-now。incumbent(CSPM/SIEM)は human-operated か、agent-embedded CLI で、この形に到達していない。

---

## 2. the pattern — あなたの "CSPM 思考" の型 `[moat:pattern]`

`推測:` 全製品で繰り返してきた型を system 規模で言語化すると、これが構想の背骨:

> **複数の決定的・交換可能な検知エンジン**(各々が GitHub の一つの逸脱面を見る)を、**一つの agentic intelligence 層(gh-intel)が Finding 契約で束ねて triage し、AI が control(是正)し、posture が全 control を可逆にする。**
> エンジンは決定的・多様・swappable(commoditize されてよい、他者製でもよい)。知能は agentic・単一・本命。GitHub の exhaust を evidence に、危険な近道(PAT・regex・無制御 enforcement)を拒否する。

過去の各論を全て subsume する:

| 旧・部分パターン | この型での位置 |
|---|---|
| thin deterministic + agentic judgment | 検知エンジン(決定的)+ gh-intel(agentic) |
| engine + thin shell(libghostty) | libverify 等が engine、gh-intel が多 engine を束ねる shell |
| semantic-as-evidence + deterministic gate | エンジン=決定的 Finding、agent=意味 triage/action 提案、auto-execute は決定的 policy で gate |
| 復旧手段がある操作は自律 / 不可逆のみ確認(本人の行動原則) | **posture が可逆性を保証する → AI が control を自律実行できる** |

---

## 3. the loop — AI-complete operations `[wedge]`

```
        GitHub (= cloud control plane: audit log / runner / PR / settings)
                                  │  exhaust
                                  ▼
   ┌───────────────── 検知エンジン群(決定的・swappable)─────────────────┐
   │  gh-intel rules │ libverify(SDLC 逸脱) │ dyscan(supply-chain, 他者) │ sisakulint(CI/CD, upstream) │
   └───────────────────────────────┬───────────────────────────────────┘
                          Finding 契約(keystone)
                                  ▼
                     ╔═══════════════════════╗   ← real-time が「実行起点(trigger)」
                     ║  gh-intel(本命)        ║
                     ║  agentic intelligence  ║──① investigate / triage(noise-killer)
                     ║  + control             ║──② 是正 action を提案
                     ╚═══════════╤═══════════╝
                  決定的 policy で gate │ (org が autonomy 閾値を設定)
                                  ▼
                       ③ control(是正)を実行 ────► GitHub 状態
                                  ▲
                  posture(宣言的 desired-state)= 復旧点
                  ④ 逸脱を desired-state へ reconcile = 全 action が可逆
```

4 つの役割と 2 つの契約:
- **検知エンジン(plural)** → Finding。real-time stream が trigger。
- **gh-intel(intelligence + control)** → triage + 是正。本命。
- **posture(libverify)** → desired-state を定義し、control を可逆に束縛する安全基盤。
- 契約 = **Finding schema**(engine→intel)+ **desired-state/posture**(control→可逆性)。

---

## 4. なぜ gh-intel が本命 `[wedge]`

`事実(確定):` intelligence 系の中で **製品は gh-intel 単品**。理由は3重の交差:

1. **real-time = trigger になれる。** snapshot(posture 周期)も CLI(agent-embedded)も実行起点になれない。audit stream だけが「今この瞬間の脅威」で AI を自律起動できる。`推測:` §3 で「latency でなく intelligence が差別化」と言ったが、より正確には **real-time の意味は「システム側の trigger になれること」**。
2. **audit log が両面を映す。** A面(GitHub-as-product: org/IAM ガバナンス)と B面(cloud 類推 = CI/CD: runner/OIDC/workflow)を同一 telemetry で見る唯一の signal。
3. **intelligence + control が一点に集まる。** triage(noise-killer)と是正が trigger 直後に連結し、loop が AI 内で閉じる。

→ **gh-intel だけが「実行起点 + 制御点」をシステム側に提供し、AI-complete 運用を成立させる。** metsuke(human-operated platform)は死に、CLI(agent-embedded)は起点になれない。

---

## 5. agent mandate — triage + control, gated, reversible `[trust]`

`事実(確定):` agent mandate = **tier-1 triage / noise-killer に control を足す**。

- agent は全 Finding を investigate、良性の大多数を evidence 付き auto-disposition、本物だけ enrich して escalate(alert fatigue を潰す)。
- **是正 action も実行する**(token 失効・PR block・runner 隔離・collaborator 剥奪 等)。
- auto-execute するかは **組織設定の決定的 policy(confidence / severity / blast-radius 閾値)で gate**。全 disposition / action は監査可能。
- **安全機構は「不可逆だから人間」ではなく「posture が可逆を保証するから AI が実行」**(§6)。autonomy 閾値は組織が調整(=ingestion cadence と同じ「組織ごとに柔軟」設計思想)。

`推測:` これは libverify thesis §6 の「intel ≠ enforcement(enforcer は audit 記録を残さない)」と矛盾しない — むしろ強化する。gh-intel の control は **detection-triggered・posture-bounded・全 log 記録**で、批判対象だった「bypass 可能・記録なしの runtime enforcement」とは別物。**検知 + 是正 + 監査記録が一つの loop に同居する**のが durability の源。

---

## 6. posture = 可逆性の基盤 `[trust / durability]`

`事実(確定):` **posture によって "あらゆるものを可逆に" するのが最終ゴール。**

- posture = libverify の verdict が定義する **desired-state(known-good baseline)**。control はそこへ reconcile するので、誤動作しても巻き戻せる(GitOps reconciliation)。
- これは本人の行動原則「**復旧手段がある操作は自律実行、不可逆な操作のみ確認**」を製品に埋め込んだもの。
- **死んだ posture(libverify)の蘇生形態:** 人間向け posture 製品(metsuke)としてではなく、①SDLC 逸脱の **検知エンジン**(§7)、②AI control を可逆に束縛する **安全基盤**、の2役で生き残る。これは [`libverify/agent-era-verification`](../../libverify/agent-era-verification/README.md) の「検証対象が変わっても engine は生き残る」の literal な実証 — PR 検証は死に、engine は detection と reversibility-substrate へ向きを変えた。

`推測:` **自動化のパラドックス**(Bainbridge "Ironies of Automation" / Google SRE 本の toil 論 / Jeffery Smith "Operations Anti-Patterns")の解:partial automation は残った手動介入を高スキル依存にして脆くする。解は loop を **AI で閉じる**こと。posture-guaranteed reversibility が「誤れば巻き戻せる」を保証するから、ルーチンは全自動・人間はスキル不要、人間介入は真に novel/不可逆な事象のみ。

---

## 7. 検知エンジン — 複数・交換可能 `[moat]`

`事実(確定):` 検知エンジンは gh-intel と**別**、かつ**複数・swappable**。gh-intel は engine 非依存。

| engine | 観点(逸脱面) | 所有 |
|---|---|---|
| gh-intel 内蔵 rule | audit log の異常(org/CI イベント) | 本人(gh-intel 同梱) |
| **libverify** | **ソフトウェア開発プロセスの逸脱**(SDLC) | 本人(検知エンジンとして導入) |
| **dyscan**(sisakudyscan) | supply-chain 改竄 | **他者**(エンジン型の実例) |
| sisakulint | CI/CD workflow の脆弱性(SAST) | upstream(ultra-supara 氏) |

`推測:` **Finding 契約が要石(keystone)** — 別実装・別言語・他者製のエンジンが gh-intel に挿さるのは、共通 Finding schema があるから。以前「`High` gap: 出力 contract 不統一」と書いたが、**それは gap ではなく構想の中心設計**に反転する。エンジンが commoditize されても・他者製でも構わない理由はここ: 価値は engine でなく intelligence 層にある。

---

## 8. moat — GitHub-native domain 深度 `[moat / durability]`

`事実(確定):` moat = **GitHub-native domain 深度**(generic SIEM / native 実装が捉えない GitHub 固有の攻撃面モデリング)。

`推測:` 静的な深度は写せるので、防御力は **深度を他者より速く更新し続ける研究装置**から来る。`事実(確定):` その装置 = **sisakuintel-pot(honeypot で一次 TTP)+ sisakuintel-worker(ecosystem scan)+ agent-idea(bug-bounty taxonomy)** — これらは**製品ではなく研究装置**(launch/販売しない)。新しい GitHub 攻撃パターンを供給し、検知エンジンの rule と agent の investigation 知識に encode される。残る durability caveat「GitHub が native で作る」には libverify thesis §6 と同じ反論(independent / cross-org / 監査記録 / 攻撃情報で先回り)。

---

## 9. delivery — 3 対称 P0 面 / org-tunable cadence `[TTFV]`

`事実(確定):`
- 配信面は **3つ全て P0・対称**: ①GitHub App(polling)②replay(CLI)③既存 SIEM 連携。会社ごとにニーズが異なり被らない。共有 engine 上の thin shell なので限界費用が低い。
- ingestion は **stream + poll を維持、頻度は組織ごとに柔軟設定**(real-time = push 即時ではなく「組織が選ぶ連続 cadence」)。

`推測:` 機能的役割は非対称になる(構想に記録): App/streaming = **trigger になれる運用面**、replay = trigger になれない **eval/TTFV 面**、SIEM = 既存 pipeline への寄生面。launch narrative 分散リスクは本人が多 segment 販売としてオーナーする。

---

## 10. falsifiers — 何が外れうるか `[durability]`

正直に。この賭けを殺し得るもの:

- `Critical` **「あらゆるものを可逆」には硬い上限がある。** detection は post-hoc — secret が既に exfiltrate された後では、rotate できても**漏洩という現実の結果は不可逆**。posture が巻き戻せるのは GitHub の **状態**であって現実の被害ではない。reversibility の射程を「状態のみ」と明示し、prevention(pre-event posture)と併用しないと過信になる。
- `High` **real-time trigger が GitHub の配信遅延に gated される。** audit stream は分単位遅延。高速攻撃には trigger が遅すぎる場合がある。webhook subset で補うか、遅延を許容する脅威モデルに絞るかの判断が要る(§9 で webhook hybrid は不採用)。
- `High` **control の信頼は一度の誤是正で失う。** posture-reversibility があっても、本番 runner 隔離や collaborator 剥奪の誤爆は運用停止級。gate policy の初期値を保守的にし、autonomy を段階開放できないと採用前に弾かれる。
- `Medium` **domain 深度 moat は GitHub の本気投資で侵食される。** 研究装置(pot/worker)の更新速度が唯一の堀。
- `Medium` **3 対称 P0 が focus を希釈する。** 多 segment 販売が成立しなければ launch がぼやける。

---

## 生死表 / scope

`事実(確定):`

| 区分 | 対象 |
|---|---|
| **本命(製品)** | **gh-intel** — 多エンジン上の agentic intelligence + control 層 |
| **検知エンジン(複数・swappable)** | libverify(SDLC 逸脱)・dyscan(他者例)・gh-intel 内蔵 rule・sisakulint(upstream) |
| **posture(安全基盤)** | libverify の verdict = desired-state / 可逆性の復旧点 |
| **研究装置(非製品)** | sisakuintel-pot・sisakuintel-worker・agent-idea |
| **死んだ posture 実験** | gh-verify・metsuke(人間運用 platform の anti-pattern) |
| **範囲外** | parsentry(コード SAST)・phishing-hunter・cve-hunter・pleno-*(別ジャンル) |

---

## References

- [`libverify/agent-era-verification`](../../libverify/agent-era-verification/README.md) — engine 生存 / 可逆 enforcement の前提論
- 本命: [gh-intel](https://github.com/HikaruEgashira/gh-intel)
- 検知エンジン: [libverify](https://github.com/HikaruEgashira/libverify) · dyscan(sisaku-security/sisakudyscan, 他者)· [sisakulint](https://github.com/sisaku-security/sisakulint)(upstream)
- 研究装置: [sisakuintel-pot](https://github.com/sisaku-security/sisakuintel-pot) · [sisakuintel-worker](https://github.com/sisaku-security/sisakuintel-worker)
- 運用論: Lisanne Bainbridge "Ironies of Automation"(1983) / Google SRE Book(toil) / Jeffery D. Smith "Operations Anti-Patterns, DevOps Solutions"

# libverify: agent-era verification

> design doc / PMF thesis。読者は自分と将来の協力者。これは「なぜこの賭けが正しいか」を
> 自問検証する思考アーティファクトであり、セールス資料ではない。
>
> 各節の `[...]` は PMF harness の次元タグ(why-now / wedge / TTFV / trust / durability / moat)。
> 関連: [`plan.md`](https://github.com/HikaruEgashira/libverify/blob/main/plan.md) /
> [ADR-0001](https://github.com/HikaruEgashira/libverify/blob/main/docs/adr/0001-verification-unit-is-agent-execution.md) /
> [ADR-0003](https://github.com/HikaruEgashira/libverify/blob/main/docs/adr/0003-behavioral-diff-deferred.md)

---

## The bet (3-sentence thesis)

**PR-ceremony は agent 時代に消えるが、「この変更は安全か」という attestable な verdict への需要は消えず、むしろ強まる。** libverify の `evidence → verdict → gate` 抽象は既に PR に依存していない(gas-check が証明 — git も PR も commit もない領域で同じエンジンが動く)ため、検証単位を「agent execution」へ移すのにエンジンの書き換えは要らない。**libverify は、runtime enforcement が構造的に残せない監査記録を産出する、独立かつ形式証明された post-execution verifier である。**

---

## 1. The wave — なぜ今 `[why-now]`

`plan.md` の前提が崩れつつある。Boris Tane「The SDLC Is Dead」(2026/02)と watany「ロボットのための工場に灯りは要らない」(2026/03)が同じ結論に達した:

- SDLC は Intent → Build → Observe に崩壊し、コードレビューは儀式になった。機械のワークフローに人間レビューを押し付けると `500 PR/day vs 10 review/day` のボトルネックになる。
- 人間レビューを前提とした工程は消え、**自動化された多層ゲート + observability** に置き換わる(swiss cheese model)。

これは libverify の 34 controls の大半(`review-independence`, `two-party-review`, `stale-review`, `branch-protection-enforcement` …)が前提する世界の終わりを意味する。検証対象(PR, branch, human reviewer)そのものが消滅する。

**事実:** 既存 controls 中、PR 消滅後も無条件に機能するのは `test-coverage`, `secret-scanning`, `vulnerability-scanning`, `security-policy`, `codeowners-coverage` 程度。

---

## 2. だが engine は生き残る `[moat:engine]`

崩れるのは controls の前提であって、エンジンの抽象ではない。

- `Control` trait は「evidence を受け取り verdict を返す」純粋な抽象。PR の存在を前提としていない。
- OPA policy engine が gate decision を宣言的に切り替える(strict / advisory)。
- Creusot 形式検証が判定述語の正しさを数学的に保証する。
- platform adapter パターンは新しい evidence source に拡張可能。

**この抽象が本当に PR 非依存である証拠が gas-check。** gas-check は Google Apps Script を検証する — git も PR も commit も存在しない領域。にもかかわらず同じ `libverify-core` の上で 18 controls が動く。これは「evidence → verdict → gate が SDLC artifact から独立している」ことの存在証明であり、agent 時代に PR-ceremony が消えてもエンジンが生き残ることの先行事例である。

→ **検証単位を commit から agent execution へ移すのにエンジン書き換えは要らない**(ADR-0001 で受理済み)。これが「波」に乗れるポジションの根拠。

---

## 3. 何を検証するか — relief-finding `[wedge / TTFV]`

### wedge は未解決である、という正直な出発点

既存の form factor(gh-verify CLI / metsuke SaaS / SARIF 出力 / 新 adapter)はどれも TTFV が遅いか価値を感じにくい。診断:

- **fast-install user ≠ value-feeling user。** 秒で `gh extension install` できる dev が得るのは process-hygiene の nag(「review がない」「未署名」)— branch protection が既に言うか、本人が所有しない事柄。verdict を value する側(auditor, security lead)は install が遅く、価値を感じるのは監査時(年1回)。
- **libverify が出すのは process hygiene の verdict。hygiene findings は最速で install する人にとって本質的に低 felt-value** — nag であって relief ではない。

### relief-finding = intent-action divergence

agent fleet を回す operator の acute pain は「**この agent がやったことを信頼できるか / 追いきれない**」。ここに刺さる単一 finding を設計する:

> **intent-action divergence** — agent の変更が stated intent を超えた / 逸脱した。
> 例:「login バグ修正を頼んだのに、billing module と CI workflow も変更した」

- **spec を事前に書く必要がない。** task そのものが intent。
- 誰もが「agent が余計なところを触った」体験を持つ。初見で「これが欲しかった」。
- GitHub diff は「何が」変わったかしか見せず、「intent と合致しているか」を誰も検証していない。incumbent 不在。
- installer = operator = value-feeler。trigger は毎実行(高頻度)。

既存 34 controls はどれもこの relief を生まない(`privileged-operation-audit` ですら blocklist = known-bad しか捕れない)。**これは新規に設計する finding。**

### TTFV を縛るのは証拠取得コスト

この finding の TTFV は「Claude Code セッションをどれだけ安く evidence(intent + action log)に変えられるか」に gated される(§7 observability が答え)。PR controls が GitHub API から無料で evidence を得るのとは異なる。

---

## 4. でも意味判断は証明できない — 決定性境界 `[trust]`

「actions が intent に合致するか」は **意味的判断**であり、deterministic predicate ではない。一方 libverify の identity ―― そして §6 で durability の根拠に選ぶもの ―― は **deterministic で Creusot 形式証明された verdict**。intent-match は形式証明できない。relief-finding が moat と正面衝突する。

**解決 = 意味判断を verdict の外に出す。不変条件「意味判断は libverify の verdict 内では決して起きない」を守る。**

```
[agent run] → semantic judge (LLM; metsuke/adapter内) → IntentConformance{
                  divergence_score, intent_summary,
                  diverged_paths, unrequested_changes, confidence }  ← evidence
            → libverify: deterministic gate(score, paths を policy で判定) → verdict (provable)
```

- 非決定的なのは **evidence**。決定的で証明されるのは **verdict**。
- ADR-0001 と整合(libverify は intercept せず、完了した execution record を evidence として消費する)。
- 「この evidence に対し gate 決定は provably correct」が保たれる。evidence の質は judge(metsuke/adapter)の責務、verdict の正しさは libverify の責務 ―― 責任が分離される。

これは新しい evidence type `IntentConformance` を `EvidenceBundle` に追加し、新 control `intent-conformance` が deterministic に gate する、という形で既存アーキテクチャに収まる。

---

## 5. でも FP は? — telemetry-fed FP-healing `[trust]`

意味 judge には false positive がある。正当な refactoring を「暴走」と誤判定すれば relief は nag に逆戻りし、gate は無視され、最後には引き剥がされる ―― Boris Tane が警告したボトルネックそのものに堕する。**relief-finding は FP で自滅する。**

静的な保守的 policy を完璧に設計しようとはしない。代わりに **telemetry を燃料とする FP-healing harness で FP→0 に高速収束させる**。

- 既存の [`fp-reduction-loop`] は **offline + synthetic**(OSSInsight trending repos を回し、TP/FP を手分類し、policy/adapter/control 層を構造修正、FP=0 まで反復)。corpus は proxy であり、production telemetry を持たない。
- ここで欲しいのは **online + telemetry-based**。natural な FP label が手元にある ―― **人間が divergent change を keep したか revert したか**。revert = 「その verdict は FP だった」。
- healing は §4 の決定性境界を壊さない。形式証明は **gate logic** を保証し、healing loop は **evidence の質と閾値の校正**を担う。役割分担。

---

## 6. なぜ durable — audit 需要 vs runtime enforcement `[durability]`

最大の反論:「runtime enforcement(pleno-cockpit 等)があれば post-hoc verifier は不要では?」

**論破の核は compliance/audit が durable 需要であること。** runtime enforcement は gate するが **attestable trail を残さない**。agent 時代に検証「対象」は変わる(PR → agent execution)が、**「この変更は安全だったか」を後から証明可能な verdict 記録への需要は不変**であり、変更頻度が上がり人間の監視が減るほど強まる。

- runtime enforcement = 実行時の gate。bypass 可能、framework 固有、記録を残さない。「agent に自分の宿題を採点させる」構造。
- libverify = 独立した system-of-record。deterministic + 形式証明された verdict + evidence を SARIF/JSON で産出。post-incident / 監査 / 規制対応はこの記録を消費する。

→ enforcement と verification は swiss cheese の別々の穴を埋める。enforcement 層に穴があっても、独立した verification 層が捕える。**verifier は enforcer の有無に関わらず、監査記録の唯一の生成源として残る。**

これが compliance-now(metsuke/vanta/drata, presets: soc2/slsa/ismap/pci-dss/…)を「単なる今日の収益」以上のものにする ―― それは agent 時代に verifier が生き残る**構造的理由**そのものである。

---

## 7. observability を first-class component に `[moat / TTFV / durability]`

§5 の telemetry は deployment の性質。SaaS だけが verdict+override を全テナント横断で見られる、とすると free CLI は healing の死に枝になる。これを避けるため **observability を libverify 自身のコンポーネントにする** ―― libverify が telemetry の **contract(OTel schema)を標準として定義し**、収集・送信は adapter(otel-hooks)/ metsuke に委ねる。`libverify-core` は純粋な判定関数のまま(常駐プロセスにはしない)。

Boris Tane の caveat「observability without action is just expensive storage」が scope を縛る。action を持つ双方向 substrate にする:

```
[agent run] ──入口①──> [evidence] ──> libverify(verdict) ──出口②──> [outcome: keep? revert?]
              OTel ingress                                OTel egress
```

| 配管 | 解決するもの |
|------|------------|
| **① 入口**(agent actions / metrics → evidence) | §3 の TTFV(証拠取得コスト)を閉じる + ADR-0003 で deferred の Layer 3 behavioral-diff を un-block(observability component が、欠けていた metrics adapter になる) |
| **② 出口**(verdict + human override → telemetry) | §5 の FP-healing flywheel を回す |

両方とも同じ OTel schema で表現できるため、別々に作るより一本の contract にする。

### moat = data flywheel

```
使用 → telemetry(verdict+override) → FP/TP label → 自動 healing → 精度↑ → 信頼↑ → 使用↑
```

engine は clone され得る。**clone されても勝てる唯一の moat は、telemetry で healing された精度。** observability を library のコンポーネントにすることで flywheel は SaaS-lock されず任意の deployment から燃料を得るが、**metsuke が最も完全な aggregator** として精度を最も速く複利化する。free CLI = 配布、metsuke = 精度が複利する場所。

---

## 8. Evidence — 4 POC が各前提を裏付ける `[wedge / moat]`

| POC | form factor | この thesis の何を証明するか |
|-----|------------|--------------------------|
| **gh-verify** | local `gh` CLI extension | wedge の配布チャネル(zero-friction install)。PR + agent-execution 両方の入口。§3 の「fast-install だが low-felt-value」現象の実物 |
| **gas-check** | local CLI / Apps Script | **§2 の核心証拠** ―― git も PR も commit もない領域で同じエンジンが動く = 抽象が SDLC artifact から独立。agent 時代に PR が消えても生き残る存在証明 |
| **atlassian-verify** | Bitbucket + Jira (private) | cross-VCS / 第三プラットフォーム。adapter パターンの一般性 |
| **metsuke** | hosted MCP server + GitHub App | §7 の flywheel aggregator。SaaS による継続監視 + agent-native(MCP)配信 + telemetry 集約。compliance-now の収益面 |

---

## 9. 何が外れうるか — falsifiers `[durability]`

正直に。この賭けを殺し得るもの:

- **`Critical` timing risk:** agent fleet を無監視で回す pain がまだ acute でない。relief-finding(intent-divergence)は今日 low felt-value で、「fleet 常態化 → 高 felt-value への flip」を待つ timing bet。flip が来なければ wedge は刺さらない。**de-risk:** §7 入口を otel-hooks に最速で繋ぎ、Claude Code セッション → verdict の TTFV を実測する。
- **`High` semantic judge の質:** divergence judge の精度が低ければ §5 healing が追いつく前に信頼を失う。**de-risk:** 初期は §4 の決定性 narrowing(`diverged_paths ∩ protected_globs` の交差時のみ block、それ以外 surface-as-review)で block を希少・高精度に保つ。
- **`High` agent framework が内製する:** Claude Code / Cursor が同等の gate を内蔵すれば independent verifier の価値が侵食される。**反論:** 「自分の宿題は自分で採点できない」(§6 独立性)+ cross-framework portability + audit 記録は framework 内蔵では残らない。最悪ケースでも libverify は彼らが embed するエンジン(libghostty model)になり得る。
- **`Medium` observability 採用:** flywheel は telemetry に依存。OTel contract が採用されなければ healing 燃料が枯れる。**de-risk:** metsuke を最初の aggregator として自前で回し、flywheel を bootstrap する。
- **`Medium` compliance 需要の前提:** 「agent 時代も attestable verdict が要る」が §6 durability の土台。規制が追いつかず誰も監査記録を求めなければ崩れる ―― が、規制は緩むより厳しくなる方向に賭ける。

---

## 10. Open decisions

`plan.md` の open decisions を、本 thesis で確定した分を反映して更新:

| # | decision | 状態 |
|---|----------|------|
| 1 | semantic 判断の配置 | **確定:** verdict の外(evidence source)。`IntentConformance` を `EvidenceBundle` に追加 |
| 2 | observability component の scope | **確定:** 双方向 OTel contract(入口=evidence/metrics, 出口=verdict+override)。core は純粋のまま |
| 3 | `IntentConformance` evidence schema | **未定:** フィールド(divergence_score, diverged_paths, unrequested_changes, confidence)の正式定義、versioning |
| 4 | 新 crate 分割 | **未定:** `libverify-observability`(OTel contract)/ semantic-judge adapter を別 crate にするか。core 非依存を保つこと |
| 5 | ADR-0003 の reversal | **未定:** observability 入口が metrics adapter を提供するなら behavioral-diff(Layer 3)を un-defer する新 ADR が要る |
| 6 | FP-healing telemetry の consent/privacy | **未定:** override label の収集に opt-in / 匿名化 / テナント分離をどう設計するか。SaaS 規約と GDPR/ISMAP 整合 |
| 7 | 閾値校正の所在 | **未定:** divergence threshold を policy(Rego)に置くか、healing loop が動的調整するか |

推奨される follow-up: 上記 1/2 を libverify 側の新 ADR(0004 semantic-as-evidence, 0005 observability-component)として記録する。

---

## References

- Boris Tane, "The Software Development Lifecycle Is Dead" (2026/02) — https://boristane.com/blog/the-software-development-lifecycle-is-dead/
- watany, "ロボットのための工場に灯りは要らない" (2026/03) — https://speakerdeck.com/watany/dark-factory-for-agent
- strongDM, "Software Factories And The Agentic Moment" — https://factory.strongdm.ai/
- Latent Space, "How to Kill the Code Review" — https://www.latent.space/p/reviews-dead
- [libverify](https://github.com/HikaruEgashira/libverify) / [gh-verify](https://github.com/HikaruEgashira/gh-verify) / [gas-check](https://github.com/HikaruEgashira/gas-check) / [metsuke](https://github.com/plenoai/metsuke) / [otel-hooks](https://github.com/HikaruEgashira/otel-hooks)

[`fp-reduction-loop`]: ~/.claude/skills/fp-reduction-loop/SKILL.md

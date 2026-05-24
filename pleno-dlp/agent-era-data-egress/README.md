# pleno: agent-era data egress control

> design doc / PMF thesis。読者は自分と将来の協力者。これは「なぜこの賭けが正しいか」を
> 自問検証する思考アーティファクトであり、セールス資料ではない。
>
> 各節の `[...]` は PMF harness の次元タグ(why-now / wedge / TTFV / trust / durability / moat)。
> 対象は [`pleno-dlp`](https://github.com/plenoai/pleno-dlp)(Go, secret+CASB scanner)と
> [`pleno-anonymize`](https://github.com/plenoai/pleno-anonymize)(Python, PII engine + LLM proxy)の2本。
> libverify / cockpit / metsuke の verification thesis とは**別系統**(egress 制御であって検証ではない)。

---

## The bet (3-sentence thesis)

**agent 時代、機密データの越境 ―― LLM provider と SaaS への egress ―― は人間のレビューでは防げなくなり、「送る前に検出して止める / 匿名化する」制御への需要が構造的に立ち上がる。** pleno はこの egress 制御を3点で差別化する:(1) **on-device 完結**(データを境界の外に出さずに検出・マスク)、(2) **日本語 PII を一級**で扱う(My Number / 在留カード / 健康保険証 ―― 既存ツールの空白)、(3) **あらゆる SaaS に拡張できる CASB 的 connector** で enterprise 導入の壁を外す。**`pleno-anonymize` が on-device の検出・匿名化エンジン(+ LLM proxy)を、`pleno-dlp` がそのエンジンを内包する secret + SaaS-CASB scanner を担い、同じ recognizer registry の上で「egress の手前」を押さえる。**

---

## 1. The wave — なぜ今 `[why-now]`

3つの波が同時に立った。

- **agent fleet による egress 量の爆発。** Claude Code 等の coding agent が日常的にソース・ログ・顧客データを LLM API へ送る。送信頻度が人間レビューの限界を超え、「誰がいつ何を外に出したか」を人が追えなくなった。**止められるのは「送る前」しかない。**
- **openai/privacy-filter の登場(2026)。** OpenAI 自身が「LLM へ渡る前の PII 除去」を製品化した ―― この問題が本物だという第三者検証。pleno はこれを競合ではなく **opt-in engine として取り込む**(`--pii-engine=openai-pf` / `--engine openai-privacy-filter`)。§6 で述べる通り、取り込むこと自体が moat の実証装置になる。
- **日本語 × 規制の空白。** Presidio / AWS Comprehend / Google DLP は英語ファースト。My Number(検査数字付き)・在留カード・健康保険証・運転免許証を一級で扱う on-device ツールが存在しない。APPI の越境移転規制が enterprise の「顧客 PII を海外 LLM / SaaS に出せない」壁を強化し、空白の価値を押し上げる。

---

## 2. wedge は enterprise CASB DLP、on-device と日本語はその成立条件 `[wedge]`

刺さりの先端は **enterprise の CASB 的 DLP** ―― SaaS(GitHub / GitLab / Bitbucket / Slack / Notion / Confluence / Jira)と filesystem / git を横断して、secret と PII を「外に出る前」に捕える層。

(1) on-device と (2) 日本語は wedge そのものではなく、**先端を成立させる差別化条件**である:

- **on-device** がなければ「DLP ツール自身が PII を境界の外に運ぶ」自己矛盾になる。enterprise が買う唯一の理由(データを出さない)を DLP が破壊する。
- **日本語** がなければ incumbent(Google DLP / Presidio)と同じ土俵に立ち、勝てない。My Number / 健康保険証を checksum 付きで一級扱いする一点が、英語ファースト勢の届かない場所をつくる。

この3つが一点に収束する form factor が CASB DLP であり、(1)(2) は単独では nag、組み合わさって初めて relief になる。

**正直な但し書き(TTFV)。** enterprise CASB DLP は wedge として felt-value は高いが install が遅い(調達・self-host・connector 設定)。これを補うのが §5 の hosted version と、`uvx pleno-anonymize` / `go install pleno-dlp` の秒速 install ―― **最初の価値創出(TTFV)を hosted と CLI で前借りし、durable な価値は on-device CASB で回収する**二段構え。

---

## 3. 2製品アーキテクチャ — engine を内包する shell `[wedge / moat]`

「1 engine 2 shells」ではない。**engine 製品と、それを内包する shell 製品**の依存関係:

```
pleno-anonymize (Python, public)              pleno-dlp (Go, private)
  PII 検出・匿名化エンジン + LLM proxy           secret scanner + SaaS-CASB scanner
  - spaCy NER (pleno_anonymize_ja/_en)          - trufflehog 互換 599 detectors
  - Presidio + regex/checksum                     (598 secrets + 2 opt-in PII)
  - /api/analyze, /api/redact                   - filesystem / git / stdin / 7 SaaS connector
  - /api/{openai,anthropic,gemini} proxy        - SARIF / CI gating / allowlist
  - SDK + CLI + WASM tokenizer                  - private-key blast radius (crt.sh CT)
        ▲                                              │
        └──── loopback HTTP supervisor ◀───────────────┘
              (--pii-engine=anonymize: pleno-dlp が
               anonymize を PII バックエンドとして spawn)
```

- **依存方向は dlp → anonymize。** pleno-dlp の PII 検出は pleno-anonymize を loopback(127.0.0.1 限定、`0.0.0.0` bind は拒否)で spawn して委譲する。secret 検出は Go 側に内蔵。
- **2言語は意図的。** secret scanning は単一 Go binary の配布容易性と速度(trufflehog 互換)が効く領域。PII NER は Python/spaCy の ML エコシステムが必要。境界は「決定的な pattern 検出(Go)」vs「学習モデル依存の意味検出(Python)」。
- **同じ recognizer taxonomy。** OPF の 8 label も pleno の `PERSON / EMAIL_ADDRESS / …` に正規化され、anonymizer / scanner / proxy が backend 非依存で同じ entity 体系を見る。

---

## 4. trust — wedge 内で強く、wedge の外で弱い `[trust]`

最大の falsifier に正面から答える。**custom JA NER は領域によって精度が大きく違う。**

| corpus | strict-span F1 | 評価 |
|---|---|---|
| in-distribution(form/record-style PII) | **≈ 0.96–0.97** | wedge のドメイン |
| narrative prose(stockmark JP Wikipedia) | **≈ 0.47** | wedge の外 |
| narrative prose(CoNLL-2003 EN) | ≈ 0.57 | wedge の外 |

これは欠陥ではなく**設計の帰結**である。enterprise CASB DLP が走査するのは SaaS のレコード・フィールド・構造化文書(Jira チケット、Notion DB、Confluence ページの表、設定ファイル)= **form-style がドメイン**。モデルは form/record に最適化され、散文には弱い。**0.47 の散文弱点は wedge の外**にある。

> 注意すべき falsifier(§9 へ):もし実トラフィックの PII が form でなく自由記述(Slack 会話の本文等)に多く宿るなら、「wedge 内のつもり」の場所で漏れが出る。domain 切り分けは measurement triad(corpus / metric / aggregation, ADR-0006)で固定し続ける。

### FP の非対称と2層対策

CASB DLP では **false-negative(PII レコード見逃し)= compliance 侵害(致命的)**、**false-positive(過剰検出)= alert 疲労 → ツール剥がし**。非対称なので両方を別の層で抑える:

1. **推論時の決定的 precision 層(即効・clone 可能)。** pleno-dlp 流にセマンティック信号を組み合わせる ―― checksum(My Number 検査数字 / Luhn / 法人番号)、keyword/context gating(regex を keyword の後ろでだけ走らせる)、severity gating、`.pleno-allow.json` allowlist(detector × raw × regex × path glob の AND、suppression 数を stderr に出して stale rule を可視化)。pleno-audit の **"default no-action / opt-in"** 原則を引き継ぎ、既定では止めず可視化から入る。
2. **継続改善 harness 層(compounding・clone 不能)。** FP signal を NER 再訓練に還す**独自の FP-reduction loop**。これは §6 の moat 本体。

### pluggable engine = trust の実証装置

OPF を opt-in で挿せる同一インターフェイスは、**「日本語 form PII では pleno でないと価値が出ない」を主張でなく benchmark で示す**装置になる。OPF(英語・散文強い)と pleno(日本語・form 強い)を head-to-head にかけ、勝ち負けを数値で出せる(`pleno-anonymize-ai-benchmark-pii-masking-300k-ja` / `pleno-anonymize-eval-ja-issue-146`)。complement を取り込みつつ自分の moat を spotlight する。

---

## 5. on-device が本体、hosted は TTFV を前借りする degraded mode `[trust / TTFV]`

差別化軸 (1) は「データを境界の外に出さない」。だが proxy は `pleno-anonymize.fly.dev` でも動く ―― そこに流せばデータは境界を越え、thesis を自己否定する。解消:

- **on-device が default かつ thesis の本体。** SDK / CLI は既定でローカル engine(builtin Presidio + `pleno_anonymize_ja` in-process、~50ms/doc CPU)。WASM tokenizer で browser-side 前処理まで完結。enterprise の本番経路は **self-host か on-device**。
- **hosted(`fly.dev`)は最初の価値創出を早めるための degraded mode。** モデル DL も self-host も要らずに `curl` 一発で試せる ―― 評価者の TTFV を最短化する playground。**本番の信頼経路ではないと明記する。**
- **LLM proxy の価値は「LLM へ出る前にローカルでマスク → 応答で復元」。** hosted proxy はこの思想の妥協版であって、思想そのものは on-device 側にある。

---

## 6. moat — 日本語 form-PII の data flywheel `[moat]`

engine も pluggable interface も clone され得る。**clone されても勝てる唯一の moat は、日本語 form-PII で実使用 / eval から複利化した精度。**

- **データ基盤:** [`0xhikae/pii-masking-300k-ja`](https://huggingface.co/datasets/0xhikae/pii-masking-300k-ja) ―― 自前で構築した日本語 PII データセット(`ai4privacy/pii-masking-300k` を起点に機械翻訳 + 擬似値合成、文字オフセット検証済み)。`pleno_anonymize_ja` の学習元。
- **measurement harness:** ADR-0006 が canonical 計測を **triad metadata(corpus / metric / aggregation)付き `scores.json`** に固定。primary metric は strict-span F1。RunPod 上で再訓練候補を回し、MERGE / COMMIT / KILL を決定的に判定する(直近イテレーションでは hf candidate を **KILL** ―― precision 8.8%(ORG)/ 44.2%(DOB)で floor 未達 ―― `pleno_ner` baseline が operational lead を維持)。**「精度が上がった」を体感でなく pre-registered な数値で言える**段階に既に到達している。
- **flywheel(未完):** この計測 harness は胚であって、まだ「実使用 / eval の FP signal を自動で再訓練に還す」閉ループにはなっていない。**pleno 専用の FP-reduction loop を新規構築する必要がある**(既存の libverify 用 `fp-reduction-loop` skill はスコープ外・offline+synthetic)。これを閉じることが moat を複利に変える条件。

```
使用 / eval → FP signal(form-PII の見逃し・過検出)→ triad で計測 → 再訓練 → 精度↑ → 信頼↑ → 使用↑
```

---

## 7. durability と GTM — bottom-up 配布 → top-down enterprise 転換 `[durability]`

wedge 先端(enterprise CASB DLP)は felt-value が高いが sales が遅い。二段モーションで橋渡しする:

- **配布(bottom-up):** dev が `uvx pleno-anonymize scan .` / `go install .../pleno-dlp` を秒で入れ、proxy は base_url 差し替えだけ。OSS + hosted お試しで TTFV を最短化。
- **転換(top-down):** enterprise が「顧客 PII を海外 LLM / SaaS に出せない」compliance 壁に当たる → **on-device 完結 + 日本語 + SaaS connector 網**が買う理由になる。
- **拡張性が壁を外す:** connector は **humble-object パターン**(pleno-dlp ADR-0003 ―― connector は HTTP/JSON orchestration の薄い値で unit test を持たず、algorithmic logic だけ helper subpackage に隔離、検証は live smoke run)で増設が安い。顧客の使う SaaS を後追いで足せ、「うちの △△ に対応してる?」を潰せる。
- **ライセンス:** 両 repo とも **AGPL-3.0**。network-copyleft が「改変して社内サービス化するなら commercial license」の転換圧をつくる(ただし §9 にこの圧の弱点 falsifier を記載)。

---

## 8. Evidence — 既に出荷済みの artifact が各前提を裏付ける `[wedge / trust / moat]`

retrospective thesis なので、賭けの前提は「これから作る」ではなく**既に動いている物**で裏付く。

| artifact | form factor | この thesis の何を証明するか |
|---|---|---|
| **LLM proxy**(`/api/{openai,anthropic,gemini}/{path}`) | hosted + self-host(v0.1.0 / PyPI 0.2.3 / fly.dev) | §1 の「送る前にマスク」が実装・出荷済み。request で redact → response で deanonymize する透過 proxy |
| **pleno-anonymize engine** | Python SDK + CLI + WASM | §2 の on-device + 日本語。builtin ~50ms/doc、checksum 付き JP entity(MY_NUMBER / RESIDENCE_CARD / HEALTH_INSURANCE / DRIVER_LICENSE …) |
| **pleno-dlp scanner** | 単一 Go binary | §2 の CASB wedge。599 detectors、7 SaaS connector、SARIF / CI gating、loopback-only PII engine、private-key blast radius(crt.sh CT 照合、private key 本体は host から出ない) |
| **pluggable OPF engine** | opt-in(`--pii-engine=openai-pf`) | §4 の trust 実証装置。complement を取り込みつつ head-to-head で moat を示す |
| **measurement harness + dataset** | `scores.json` triad / `pii-masking-300k-ja` / benchmark・eval repo | §6 の moat の胚。pre-registered な数値で精度を語れる |

---

## 9. 何が外れうるか — falsifiers `[durability / trust]`

この賭けを殺し得るもの。正直に。

- **`Critical` 散文 PII 漏れ:** wedge を「form-style」と定義したが、実トラフィックの PII が自由記述(Slack 会話本文、議事録、問い合わせメール)に多く宿るなら、real prose F1 0.47 のまま false-negative=漏れが出て、compliance を裏切る。**de-risk:** §4 の2層 FP対策 + form/prose の domain 切り分けを measurement triad で固定し、wedge の境界を数値で監視し続ける。
- **`High` AGPL の転換力が弱い:** AGPL copyleft は主に「改変して第三者にネットワーク提供」で発火する。enterprise が **未改変のまま社内 self-host** する限り commercial license を買う強制力が弱く、monetization engine が空振りしうる。**de-risk:** monetization 軸を AGPL 一本に賭けず §10 の2軸を保持。
- **`High` 大手の追随:** Google DLP / AWS / Presidio が日本語 form-PII を本気で埋める、または OpenAI が privacy-filter を多言語化する。JA flywheel が追いつかれる。**反論:** §6 の data flywheel を先に閉じて精度を複利させ、追随コストを上げる。日本語 form-PII の eval corpus を持っているのは pleno 側。
- **`Medium` FP-reduction loop 未構築:** 独自 loop を作り切れず、計測 harness が胚のまま閉ループにならない → moat が育たない。**de-risk:** ADR-0006 の triad 計測を起点に、eval signal → 再訓練の自動化を最優先で閉じる。
- **`Medium` enterprise が on-device を嫌う:** self-host 運用負荷を嫌い hosted を求める → on-device という最大の差別化が需要と噛み合わない。**反論:** それでも「データが境界を越える」事実は変わらず、compliance 部門が self-host を要求する側に賭ける。hosted は TTFV 用の degraded mode に留める。

---

## 10. Open decisions

| # | decision | 状態 |
|---|----------|------|
| 1 | wedge 先端 | **確定:** enterprise CASB DLP。on-device(1)/ 日本語(2)は成立条件 |
| 2 | on-device vs hosted | **確定:** on-device=本体・durable 経路、hosted=TTFV を前借りする degraded mode |
| 3 | OPF の位置づけ | **確定:** 競合でなく opt-in complement、かつ moat の実証装置 |
| 4 | FP 対策 | **確定(設計):** 推論時の決定的 precision 層 + 継続改善 harness 層の2層 |
| 5 | **独自 FP-reduction loop** | **未構築:** eval/使用 FP signal → 再訓練の閉ループを pleno 専用に新規構築する(現状は ADR-0006 の triad 計測=胚) |
| 6 | **monetization** | **未定(2軸保持):** (a) AGPL → commercial license、(b) hosted SaaS / 管理コンソール。どちらを主軸にするか未決 |
| 7 | wedge 境界の数値監視 | **未定:** real prose vs form の混在比を実トラフィックで計測し、§9 Critical falsifier を早期検知する仕組み |

---

## References

- [`pleno-dlp`](https://github.com/plenoai/pleno-dlp) — Go secret + CASB scanner / [ADR-0003 connectors as humble objects](https://github.com/plenoai/pleno-dlp/blob/main/docs/adr/0003-connectors-as-humble-objects.md)
- [`pleno-anonymize`](https://github.com/plenoai/pleno-anonymize) — Python PII engine + LLM proxy / [ADR-0006 measurement triad](https://github.com/plenoai/pleno-anonymize/blob/main/docs/adr/0006-supersede-0005-with-phase2-numbers.md) / [ADR-0003 spaCy-LLM + Presidio](https://github.com/plenoai/pleno-anonymize/blob/main/docs/adr/0003-spacy-llm-presidio.md)
- [`pleno-anonymize-ai-benchmark-pii-masking-300k-ja`](https://github.com/plenoai/pleno-anonymize-ai-benchmark-pii-masking-300k-ja) / [`pleno-anonymize-eval-ja-issue-146`](https://github.com/plenoai/pleno-anonymize-eval-ja-issue-146) — moat の実証 harness
- [`0xhikae/pii-masking-300k-ja`](https://huggingface.co/datasets/0xhikae/pii-masking-300k-ja) — 日本語 PII データセット(moat のデータ基盤)
- [`openai/privacy-filter`](https://github.com/openai/privacy-filter) — opt-in complement engine
- [`pleno-audit`](https://github.com/plenoai/pleno-audit) — local-first / no-DB / opt-in 原則の出所(browser-side CASB の姉妹)
- Microsoft [Presidio](https://github.com/microsoft/presidio) — PII 検出フレームワーク基盤

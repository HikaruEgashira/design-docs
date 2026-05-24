# design-docs

あらゆる OSS の設計ドキュメントを集約する repo。`{repo}/{topic}/README.md` 規約。

## Harness: PMF engineering

**Goal:** 新規 OSS が「刺さる(PMF)」状態で設計・ローンチされるよう、設計時と launch 前に PMF を採点する。

**Trigger:** OSS の design doc を書く / ローンチ前にレビューするときは、対象 repo で `pmf-audit` skill を使う(`.claude/settings.json` で `pmf@hikae-marketplace` を有効化済み)。TTFV / 信頼(false-positive economics)/ wedge の3次元を採点し、合成 PMF readiness と優先度付き改善提案を出す。単純な質問は直接答えてよい。

harness 本体は [`HikaruEgashira/agent-skills` の `pmf` plugin](https://github.com/HikaruEgashira/agent-skills/tree/main/pmf)。

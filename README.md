# design-docs

Design documents for my OSS. These are snapshots — they are not maintained
continuously, so they may be stale.

## Why managed here?

各 OSS の repo 内で設計ドキュメントを管理したくない。新機能の実装時や過去を振り返るときに
役立つが、コードと寿命が違うため別の場所にまとめる。

ディレクトリ規約は `{repo}/{topic}/README.md`。1 OSS = 1 ディレクトリ、トピック単位で design doc を置く。

## 新規 OSS を設計するとき

この repo は PMF エンジニアリングの監査 harness を `.claude` で有効化している
([`pmf` plugin](https://github.com/HikaruEgashira/agent-skills/tree/main/pmf))。
新しい OSS の design doc を書く・ローンチ前にレビューするときは、対象 repo で `pmf-audit` を回し、
TTFV / 信頼(false-positive economics)/ wedge の3次元で「刺さるか」を採点する。
詳細は [`CLAUDE.md`](CLAUDE.md)。

## Design Docs

<!-- 各 OSS の design doc をここに索引する。例:
- [gh-verify](https://github.com/HikaruEgashira/gh-verify)
  - [Indeterminate verdict の設計](gh-verify/verdict/README.md)
-->

_(まだありません。最初の design doc を書いたらここに索引する。)_

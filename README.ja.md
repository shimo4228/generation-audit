Language: [English](README.md) | 日本語

# generation-audit

[![Ask DeepWiki](https://deepwiki.com/badge.svg)](https://deepwiki.com/shimo4228/generation-audit)

新しい Claude のモデルがハーネスの役（判断役・build 役・日常のセッション）に就いたときに、自作の Claude Code 資産（rules / CLAUDE.md / output style / skills / agents）を監査する [Agent Skill](https://agentskills.io/specification)（英語）です。旧いモデルの弱点を補うために書いたルールは、次のモデルでは**衝突コスト**、つまりモデルが毎リクエスト黙って解決させられる矛盾した指示に変わることがあります。このスキルは 1 回の実行でそれを見つけ、安全に直せる行は直し、残りは証拠を添えて承認を求めるか、stocktake スキルに渡します。

[Agent Knowledge Cycle (AKC)](https://github.com/shimo4228/agent-knowledge-cycle) の **Scaffold Dissolution** における、モデル世代交代トリガーの再実行できる手順です。

## 1 回の実行で 2 つの検査

| 検査 | 照らす相手 | エンジン |
|---|---|---|
| runtime 照合（Phase 1–2） | 推論時に実際にロードされている system prompt と tool description | このスキル |
| dated pattern 走査（Phase 3） | 対象モデルの公式ガイドが挙げる、古くなった書き方 | `/claude-api prompt-audit`（Claude Code に同梱の Claude API スキル） |

1 つ目は、汎用のスキャナが行わない検査です。runtime 層の一部は設定リポジトリの外から注入されるので、リポジトリだけを見ても分かりません。2 つ目はあえて委ねています。パターン表は Claude Code に同梱の `claude-api` スキルの中にあり、Anthropic が新しいモデルの公開に合わせて改訂します（CLI 2.1.280 で確認）。写しを持つと次の公開で古びるので、実行のたびに読みます。

## 手順

0. **Phase 0 — 対象と範囲**: 役に就いたモデル（判断役・build 役・日常のセッション）と、その役が読む資産を決めます。外部スキルの未改変の写しと、別のセッションが編集中のファイルは、監査はしますが変更しません。
1. **Phase 1 — runtime 層の採取**: 対象のモデルを実際の環境で動かしたセッションで、テーマを 1 つずつ逐語で引用させます。検出ゼロは「この質問では見つからなかった」という意味で、「競合がない」という意味ではありません。
2. **Phase 2 — 分類**: 食い違いを**競合**（資産と runtime 層が同時にロードされたまま食い違う）と**冗長**（runtime 層がすでに同じことを言っている）に分けます。もう存在しない設定やツールへの幽霊参照も競合に数えます。
3. **Phase 3 — dated pattern の走査**: `/claude-api prompt-audit` を、対象を分けて read-only の subagent で並列に回します。
4. **Phase 4 — 判定と適用**: 各件を 4 つの問いで判定します。意図（上書きは意図的か）、根拠（理由の記録は残っているか）、鮮度（前提はこのモデルでも成り立つか）、失効条件（宣言したトリガーは発火したか）です。dated pattern の所見のうち確度 High と Medium は、行の修正として適用します。修正まで頼まれていればそのまま、そうでなければグループごとに diff を示し、承認を得てから適用します。常時ロードされるファイル（rules・CLAUDE.md・output style）は、どちらの場合も 1 件ずつ diff を示します。
5. **Phase 5 — 委譲と記録**: runtime 照合の所見と、資産ごと消える・統合される候補はここに来ます。CLAUDE.md と output style は、このスキルが 1 件ずつ承認を得て直します。rules・skills・agents は、証拠として資産の種類ごとの stocktake スキルに渡し、処分はそちらが決めます。適用した行は commit の本文に記録します。

## 手順に織り込んだ落とし穴

- **「検証ステップは削れ」の誤診**: 公式が対象にしているのは*モデルの自己検証*です。機械が実行する決定論的な検査（build・tests・hooks）は対象外で、このスキルは両者を区別します。
- **削除ではなく反転**: 向きが変わった指示（確信度のしきい値で指摘を抑える節など）は、逆向きに書き直す必要があります。消すだけでは抑制の枠組みが残ります。

## 前提と、自分のハーネスで使うとき

- **Claude Code** と、同梱の `claude-api` スキルが必要です。Phase 3 はそのスキルのベースディレクトリから `shared/prompt-audit.md` と `shared/model-migration.md` を読みます。この場所は CLI の版で変わります。
- **著者のハーネス向けに書かれています。** 本文は `~/.claude` の path（`rules/common/akc-cycle.md`、`scripts/hooks/harness_lint.py`、`skill-health` の参照スキャナ、`skill-creator` の書き方の規則）を名指しし、判断の記録を番号（ADR-00NN。[claude-harness](https://github.com/shimo4228/claude-harness/tree/main/docs/adr) で公開）で引いています。別のハーネスでは、自分の lint・参照チェッカー・書き方の規約に置き換えてください。Phase 0–5 の流れはそれらに依存しません。

## インストール

### Claude Code

```bash
cp -r skills/generation-audit ~/.claude/skills/generation-audit
```

### SkillsMP

```bash
/skills add shimo4228/generation-audit
```

## 使い方

```
/generation-audit        # 新しい Claude のモデルがハーネスの役に就いたときに実行
```

モデルが自分から起動することはありません（`disable-model-invocation: true`）。世代交代は稀で明示的な出来事なので、確率的な自発起動には頼りません。

## コンパニオンスキル

Phase 5 は、rules・skills・agents についての所見を 3 つの stocktake スキルに渡します。無くても dated pattern の修正は適用され、runtime 照合の所見もチャットに一覧されますが、それらの資産の処分は決まりません。ループ全体は次の 3 つを前提にしています。

- [`rules-stocktake`](https://github.com/shimo4228/rules-stocktake) — 毎セッション読み込まれる rules
- [`skill-stocktake`](https://github.com/shimo4228/skill-stocktake) — 導入済みの skills
- [`agent-stocktake`](https://github.com/shimo4228/agent-stocktake) — agent 定義

## 出自

runtime 照合は、Claude 4 → Claude 5 の世代交代（2026-07）で行った監査から来ています。この監査で常駐する rules は 5,789 語から 2,314 語に減りました。見つかったものには、新しい `EnterPlanMode` の tool description と正面から食い違う plan mode の禁止、もう存在しない設定キーへの幽霊参照、agent 定義の中の確信度のしきい値で指摘を抑える節があります。dated pattern の走査は Fable 5.1（2026-09-02、High 3 件・Medium 85 件）で初めて回しました。Opus 5.5（2026-09-25）で 2 つをこの単一の入口にまとめました。この回の dated pattern 走査は 107 件を見つけ、64 件を適用しました。Opus 5.5 に固有の型は 0 件で、大半は資産に残った経緯の語りと版に縛られた書き方でした。同じ回の runtime 照合は略式で行い、意図的な上書き 1 件を見つけてそのまま残しました。

## このスキルについて

[Agent Knowledge Cycle (AKC)](https://github.com/shimo4228/agent-knowledge-cycle)（[DOI 10.5281/zenodo.19200726](https://doi.org/10.5281/zenodo.19200726)）の Scaffold Dissolution における、モデル世代交代トリガーの実装です。

## License

MIT

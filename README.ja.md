Language: [English](README.md) | 日本語

# generation-audit

[![Ask DeepWiki](https://deepwiki.com/badge.svg)](https://deepwiki.com/shimo4228/generation-audit)

新しい Claude のモデルがハーネスの役（仕事を渡して受け入れる判断役、それを実装する build 役、日常のセッション）に就いたときに、自作の Claude Code 資産（rules / CLAUDE.md / output style / skills / agents）を監査する [Agent Skill](https://agentskills.io/specification)（英語）です。旧いモデルの弱点を補うために書いたルールは、次のモデルでは**衝突コスト**、つまりモデルが毎リクエスト黙って解決させられる矛盾した指示に変わることがあります。このスキルは 1 回の実行でそれを見つけ、安全に直せる行は直し、残りは証拠を添えて承認を求めるか、rules・skills・agents を点検するスキル（[連携するスキル](#連携するスキル) 参照）に渡します。

行の修正をそのまま適用するのは、依頼が修正まで含むとき（「直して」「仕上げて」）だけです。そうでなければグループごとに diff を示して承認を得てから適用し、常時ロードされるファイルはどちらの場合も 1 件ずつ diff を示します。関連する記事とリポジトリは [著者のほかの仕事](#著者のほかの仕事) にまとめています。

## 1 回の実行で 2 つの検査

| 検査 | 照らす相手 | エンジン |
|---|---|---|
| runtime 照合（Phase 1–2） | runtime 層、つまり推論時に実際にロードされている system prompt と tool description | このスキル |
| dated pattern 走査（Phase 3） | 対象モデルの公式ガイドが挙げる、古くなった書き方 | `/claude-api prompt-audit`（Claude Code に同梱の Claude API スキル） |

1 つ目は、汎用のスキャナが行わない検査です。runtime 層の一部は設定リポジトリの外から注入されるので、リポジトリだけを見ても分かりません。2 つ目はあえて委ねています。パターン表は Claude Code に同梱の `claude-api` スキルの中にあり、Anthropic が新しいモデルの公開に合わせて改訂します（Opus 5.5 の回は CLI 2.1.280 に同梱の版を読みました）。写しを持つと次の公開で古びるので、実行のたびに読みます。

たとえば、この手順の出自である Claude 4 → Claude 5 の監査では、あるルールが「直接的な実装指示には即座に実行する。plan mode に入らない」と書いていました。同じコンテキストにロードされていた `EnterPlanMode` の tool description は "Prefer using EnterPlanMode for implementation tasks unless they're simple."（実装タスクには、単純なものを除き EnterPlanMode を優先せよ）と言っていました。この組が競合です。ルールの前提だった「計画せずに走り出す旧いモデル」はもう成り立たないので、このルールは退役させました。

## インストール

generation-audit は修正を適用した後に [skill-health](https://github.com/shimo4228/skill-health)（英語）の参照スキャナ（スキルが名指しするスクリプト・ファイル・兄弟スキルが今もあるかを確かめるもの）を実行するので、2 つは一緒にインストールしてください。スキャナは [`uv`](https://docs.astral.sh/uv/) と Python 3.11 以上で動きます。

インストールの方法は 2 つあります。clone する方法（1 つ目のブロック）ではこの 2 つだけが入ります。akc-cycle プラグイン（2 つ目のブロック）では、2 つに加えて、所見を受け取り rules・skills・agents のどれを残し、統合し、退役させるかを決める 3 つの stocktake スキルも入ります（[連携するスキル](#連携するスキル) 参照）。

```bash
git clone https://github.com/shimo4228/generation-audit
git clone https://github.com/shimo4228/skill-health
mkdir -p ~/.claude/skills
cp -r generation-audit/skills/generation-audit skill-health/skills/skill-health ~/.claude/skills/
```

clone で入れた場合、スキルはスキャナを `~/.claude/skills/skill-health` から呼ぶので、フォルダはこの場所に置いてください。新しい Claude のモデルがハーネスの役に就いたときに実行します。runtime 層の採取（Phase 1）で新しいモデルに与えられている指示を引用させるので、そのモデルをその役が働く環境で動かしているセッション（著者のハーネスでは build 役が cloud のセッションで動くため、build 役が替わったときは cloud のセッション）から実行してください:

```
/generation-audit
```

モデルが自分から起動することはありません（`disable-model-invocation: true`）。世代交代は稀で明示的な出来事なので、確率的な自発起動には頼りません。

同じスキルは、Agent Knowledge Cycle（AKC: コーディングエージェントが繰り返した経験をスキルとルールに変える、人が承認する著者の 6 フェーズのサイクル）のほかのスキルと一緒に、Claude Code プラグイン [akc-cycle](https://github.com/shimo4228/akc-cycle)（英語）にも入っています。generation-audit はこのサイクルの Curate（整理）フェーズ、つまり溜まったものを監査するフェーズに属し、プラグインでは `/akc-cycle:generation-audit` という名前で呼びます。このリポジトリは同じ元から一方向に同期しているので、同期と同期の間はプラグインより古いことがあります。

```
/plugin marketplace add shimo4228/akc-cycle
/plugin install akc-cycle@akc-cycle
```

## 前提と、自分のハーネスで使うとき

- **Claude Code** と、同梱の `claude-api` スキルが必要です。古くなった書き方の検査は、そのスキルのベースディレクトリから `shared/prompt-audit.md` と `shared/model-migration.md` を読みます。この場所は CLI の版で変わるので、スキルは実行のたびに `/claude-api` が示す場所から読みます。
- **著者のハーネス向けに書かれています。** 本文は `~/.claude` の path（`rules/common/akc-cycle.md`、`scripts/hooks/harness_lint.py`、`skill-creator` の書き方の規則）を名指しし、判断の記録を番号（ADR-00NN。[claude-harness](https://github.com/shimo4228/claude-harness/tree/main/docs/adr) で公開）で引いています。版に縛られた書き方を止める `harness_lint.py` は公開されていません。別のハーネスでは、これと書き方の規則を自分の lint と規約に置き換え、上で入れた `skill-health` の参照スキャナはそのまま使うか置き換えてください。監査の手順そのものはそれらに依存しません。

## 手順

0. **Phase 0 — 対象と範囲**: 役に就いたモデル（判断役・build 役・日常のセッション）と、その役が読む資産を決めます。外部スキルの未改変の写しと、別のセッションが編集中のファイルは、監査はしますが変更しません。
1. **Phase 1 — runtime 層の採取**: 対象のモデルを実際の環境で動かしたセッションで、テーマを 1 つずつ逐語で引用させます。検出ゼロは「この質問では見つからなかった」という意味で、「競合がない」という意味ではありません。
2. **Phase 2 — 分類**: 食い違いを**競合**（資産と runtime 層が同時にロードされたまま食い違う）と**冗長**（runtime 層がすでに同じことを言っている）に分けます。もう存在しない設定やツールへの幽霊参照も競合に数えます。
3. **Phase 3 — dated pattern の走査**: `/claude-api prompt-audit` を回します。範囲が大きいときは、対象を分けて read-only の subagent で並列に回します。各所見は `file:line`、引用した行、パターン、対象モデルにとってなぜ古いか、確度、対処の形で出ます。
4. **Phase 4 — 判定と適用**: 各件を 4 つの問いで判定します。意図（上書きは意図的か）、根拠（理由の記録は残っているか）、鮮度（前提はこのモデルでも成り立つか）、失効条件（宣言したトリガーは発火したか）です。dated pattern の所見のうち確度 High と Medium は、行の修正として適用します。依頼が修正まで含んでいればそのまま、そうでなければグループごとに diff を示し、承認を得てから適用します。確度 Low と `flag` の所見は、チャットに出す台帳に載せるだけです。常時ロードされるファイル（rules・CLAUDE.md・output style）は、どちらの場合も 1 件ずつ diff を示します。
5. **Phase 5 — 委譲と記録**: runtime 照合の所見と、資産ごと消える・統合される候補はここに来ます。CLAUDE.md と output style は、このスキルが 1 件ずつ承認を得て直します。rules・skills・agents は、証拠として資産の種類ごとの stocktake スキルに渡し、処分はそちらが決めます。適用した行は commit の本文に記録します。

## 手順に織り込んだ落とし穴

- **「検証ステップは削れ」の誤診**: 公式が対象にしているのは*モデルの自己検証*です。機械が実行する決定論的な検査（build・tests・hooks）は対象外で、このスキルは両者を区別します。
- **削除ではなく反転**: 向きが変わった指示（確信度のしきい値で指摘を抑える節など）は、逆向きに書き直す必要があります。消すだけでは抑制の枠組みが残ります。

## 連携するスキル

Phase 5 は、rules・skills・agents についての所見を 3 つの stocktake スキルに渡します。無くても dated pattern の修正は適用され、runtime 照合の所見もチャットに一覧されますが、それらの資産の処分は決まりません。ループ全体は次の 3 つを前提にしています。

- [`rules-stocktake`](https://github.com/shimo4228/rules-stocktake): 毎セッション読み込まれる rules と、それらが払わせるコスト
- [`skill-stocktake`](https://github.com/shimo4228/skill-stocktake): 導入済みの skills
- [`agent-stocktake`](https://github.com/shimo4228/agent-stocktake): agent 定義

## 著者のほかの仕事

- **[Opus 5 世代でルールの書き方は公式に変わった——自作ルールの棚卸し手順](https://zenn.dev/shimo4228/articles/claude5-rules-official-shift-audit)**（[English](https://dev.to/shimo4228/opus-5-changed-how-rules-should-be-written-audit-yours-4fb4)）: このスキルが一般化した、手で回した監査です。runtime 層を採取し、各ルールをそれと照らし、残す・直す・退役させるを決めています。
- **[Agent Knowledge Cycle](https://github.com/shimo4228/agent-knowledge-cycle)**: Curate を含むサイクルの各フェーズがなぜあるのかを、日付付きの設計判断として記録しています。
- **[harness-pruning](https://github.com/shimo4228/harness-pruning)**: 著者のハーネスから退役させたスキル・ルール・agent・hook を、それぞれの判断と一緒に日付付きで記録しています。
- **[shimo4228](https://github.com/shimo4228/shimo4228)**: 著者の拠点リポジトリです。AKC をほかの長期プロジェクトとその DOI と並べています。

## ライセンス

MIT

<details>
<summary>ツールと AI アシスタント向けの資料</summary>

generation-audit は、新しい Claude のモデルがハーネスの役（判断役・build 役・日常のセッション）を引き継いだときに、自作の rules・CLAUDE.md・output style・skills・agents を点検し直す Claude Code 向けの Agent Skill です。旧いモデルを補うために指示を書いてきた人が、そのうちどれが新しいモデルに与えられる指示と食い違うか、あるいは同じことを繰り返しているだけかを知るためのものです。安全に直せる行は直し、資産ごとの処分は証拠を添えて stocktake スキルに渡します。

存在する理由は、旧いモデルの弱点を補うために書いたルールが、次のモデルでは衝突コスト、つまり同じコンテキストに同時に読み込まれ、モデルが毎リクエスト黙って解決させられる矛盾した指示に変わることです。推論時にモデルに与えられる指示の一部は設定リポジトリの外から注入されるので、このスキルは対象のモデルが実際に読み込む system prompt と tool description を逐語の引用として採取して資産と照らし、dated pattern の検査は Anthropic 自身の `/claude-api prompt-audit` に委ねて、パターン表を写して古びさせることをしません。このスキルは、AKC の Scaffold Dissolution（モデルが自分でできるようになった仕事についてのハーネス側の指示は消す、という規則）を、モデルの世代交代のたびに再実行するための手順です。

基本的な事実: MIT ライセンス。スキル本体（`skills/generation-audit/`）はスクリプトのない `SKILL.md` 1 枚です。著者 1 人（@shimo4228）が保守しています。状態: 稼働中で、著者の Claude Code ハーネスから `scripts/sync-from-local.sh`（commit はしません）で一方向に同期しており、akc-cycle プラグインにも `/akc-cycle:generation-audit` として入っているため、同期と同期の間はこのリポジトリがプラグインより古いことがあります。要件: 同梱の `claude-api` スキルを持つ Claude Code、修正後の参照スキャン用に `~/.claude/skills/skill-health` にインストールした skill-health と `uv`。有料の鍵は不要ですが、範囲が大きいときに Phase 3 で並列に回す read-only の subagent は Claude の利用枠を使います。`/generation-audit` と呼んだときだけ動きます（`disable-model-invocation: true`）。著者のハーネス向けに書かれていて、`~/.claude` の path と著者の未公開の `scripts/hooks/harness_lint.py` を名指しし、claude-harness で公開している ADR を引いています。6 つのフェーズは、0 対象と範囲、1 runtime 層の採取、2 競合か冗長かの分類、3 dated pattern の走査、4 判定（意図・根拠・鮮度・失効条件）と確度 High/Medium の行の修正の適用、5 rules は rules-stocktake・skills は skill-stocktake・agents は agent-stocktake への委譲と、適用した行の commit 本文への記録です。

例: Opus 5.5 の回（2026-09-25）では、dated pattern の走査が 107 件を見つけて 64 件を適用しました。ガイドが Opus 5.5 で新しく挙げる型に当たるものは 0 件で、所見は Opus 5 から引き継がれた型、主に経緯の語りと版に縛られた書き方でした。runtime 照合の出自である Claude 4 → Claude 5 の監査（2026-07）は、常駐する rules を 5,789 語から 2,314 語に減らし、新しい `EnterPlanMode` の tool description と食い違う plan mode の禁止を見つけました。Opus 5.5 の回の runtime 照合は略式で行い、意図的な上書き 1 件を見つけてそのまま残しました。

リンク: [skills/generation-audit/SKILL.md](skills/generation-audit/SKILL.md)（英語）がスキル本体、[llms.txt](llms.txt)（英語）が機械可読の要約、[CHANGELOG.md](CHANGELOG.md)（英語）がリリース履歴、判断の記録は [claude-harness docs/adr](https://github.com/shimo4228/claude-harness/tree/main/docs/adr) にあります。このスキルは [Agent Knowledge Cycle (AKC)](https://github.com/shimo4228/agent-knowledge-cycle) の Curate フェーズの一部を実装しています。AKC の concept DOI（常に最新版へつながる代表 DOI）は [10.5281/zenodo.19200726](https://doi.org/10.5281/zenodo.19200726) で、引用はこの DOI で行ってください。サイクル全体をインストールできる形は [akc-cycle](https://github.com/shimo4228/akc-cycle)（英語）です。

</details>

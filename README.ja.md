Language: [English](README.md) | 日本語

# generation-audit

自作の Claude Code 資産（rules / CLAUDE.md / skills / agents）を対象に、**モデル世代交代時の監査**を実行する [Agent Skill](https://agentskills.io/specification) です。新しいモデル世代が出ると、旧世代の弱点を補うために書いたルールは**衝突コスト**——モデルが毎リクエスト黙って解決させられる矛盾指示——に転じえます。このスキルはそれを見つける手順の正本です。

[Agent Knowledge Cycle (AKC)](https://github.com/shimo4228/agent-knowledge-cycle) の **Scaffold Dissolution** におけるモデル世代交代トリガーの、再実行可能な具体手順です。

## 核となる発想 — 実際にロードされているものと照合する

照合の正本は公式ブログやドキュメントではありません。推論時に自作資産と context window を共有している **runtime 層**——system prompt と tool description——です。手順:

1. **Phase 1 — runtime 層の採取**: 実セッションからテーマ別に逐語引用で採ります（自己申告の限界と別セッション再現確認を組み込み済み）。設定リポジトリだけ見ても全体は分かりません——runtime 層はその外からも注入されます。
2. **Phase 2 — 3 分類**: **競合**（同一 context に同時ロードされた矛盾指示）/ **冗長**（substrate が既に言っている）/ **ドリフト**（ロードされない公式推奨からの乖離）。
3. **Phase 3 — 4 観点判定**: 意図（上書きは意図的か）/ 根拠（why の記録は残っているか）/ 鮮度（前提は今も成立するか）/ 失効条件（宣言済みトリガーは発火したか）。「競合 = 悪」の機械適用は明示的に禁止しています——意図的な上書きは存在します。
4. **Phase 4 — 証拠の委譲**: このスキルは **verdict を持ちません**。証拠台帳を資産クラス別の stocktake スキルに渡し、verdict 表と 1 件ずつの確認フローはそちらが持ちます。

## 手順に織り込んだ落とし穴

- **「検証ステップは削れ」の誤診** — 公式が対象とするのは*モデルの自己検証*であり、決定論的な機械チェック（build / tests / hooks）ではありません。スキルは両者を区別します。
- **削除でなく反転** — 方向が変わった指示（確信度しきい値の抑制節など）は逆向きに書き直す必要があります。削除では抑制のフレームが残ります。

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
/generation-audit        # 新しいモデル世代が出たときに実行
```

意図的に `user-invocable` にしてあります——世代交代は稀で明示的なイベントであり、確率的な自発トリガーには頼りません。

## コンパニオンスキル

Phase 4 は verdict を 3 兄弟の stocktake に委譲します。無くても動きます（証拠台帳までは生成されます）が、フルループは以下を想定しています:

- [`rules-stocktake`](https://github.com/shimo4228/rules-stocktake) — 常駐 rules（residency cost model）
- [`skill-stocktake`](https://github.com/shimo4228/skill-stocktake) — 導入済み skills（trigger-pollution cost model）
- [`agent-stocktake`](https://github.com/shimo4228/agent-stocktake) — agent 定義（ハイブリッド cost model）

## 出自

この手順は Claude 4 → Claude 5 世代交代（2026-07）で実施した実監査の一般化です。全ルールを runtime 層と照合・分類・処分し、常駐を 5,789 → 2,463 words に削減しました——新しい `EnterPlanMode` の tool description と正面衝突していた plan mode 禁止ルール、もう存在しない settings キーへの幽霊参照、agent 定義内の確信度しきい値による抑制節、などを含みます。

## このスキルについて

[Agent Knowledge Cycle (AKC)](https://github.com/shimo4228/agent-knowledge-cycle)（[DOI 10.5281/zenodo.19200726](https://doi.org/10.5281/zenodo.19200726)）の Scaffold Dissolution・モデル世代交代トリガーの実装です。

## License

MIT

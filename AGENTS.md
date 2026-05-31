# AGENTS.md

このリポジトリは **AI コーディングエージェントが Python プロジェクトを書くための Agent Skills 集** です。

## 配布形式

スキルは [Agent Skills 仕様](https://cli.github.com/manual/gh_skill_install) に従い、`gh skill install` でインストールされる前提:

```bash
gh skill install bayashi-cl/my-python-skills <skill-name>
```

自動検出パターンは `skills/*/SKILL.md`。インストール時にソース追跡用メタデータが frontmatter に注入されるため、こちらで `source:` 等の追跡フィールドを書く必要はない。

## Agent Skillsの仕様

このレポジトリで作成するSkillは<https://agentskills.io>の仕様に従います。

## スキルを追加するとき

1. `skills/<kebab-case-name>/SKILL.md` を作る (1 スキル = 1 ディレクトリ)。
1. frontmatter 必須項目: `name` (ディレクトリ名と一致) と `description` (エージェントがスキル選択に使う一文 — *いつ* 起動すべきかが分かる粒度で書く)。
1. 本文は対象エージェントへの指示として書く。「Python プロジェクトを書く」という用途に閉じているので、汎用的な Python の作法ではなく、このスキルが解決する特定のタスク (例: パッケージ初期化、テスト雛形、特定ライブラリの使い方) にスコープを絞る。
1. スキルが参照するテンプレートやスクリプトは同じディレクトリ配下に置く。

## このリポジトリで *やらない* こと

- スキル横断の共有ライブラリ化 — 各 `skills/<name>/` は自己完結させ、他スキルへの相対参照を作らない (`gh skill install` は単一スキル単位で配布される)。

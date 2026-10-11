---
paths:
  - "Makefile"
  - "scripts/**"
description: >
  Makefile・scripts/ を編集する際の方針。Makefile は scripts/*.sh を 1 行で呼ぶだけの薄い入口とし、
  処理の定義は scripts/ にのみ置く。ターゲット追加・改名・削除の手順、各ターゲットの契約、
  新規スクリプトの規約を規定する。scripts/hooks/** の書き方は git-hooks.md の担当。
---

# Makefile・scripts 編集ルール

## 目的

ローカル（`make <target>`）・Git hooks（`lefthook.yml`）・CI（`.github/workflows/ci.yml`）は同じコマンド契約を
共有する。入口は複数でも処理の定義は `scripts/` の 1 か所に置き、Makefile・CI・hooks で二重定義しない。
一般的な設計指針は make スキル（`.claude/skills/make/SKILL.md`、特に `references/architecture/command-contracts.md`）を参照する。

`scripts/hooks/**` の書き方（fail-closed・ロケール・`git diff` の扱い等）は `.claude/rules/git-hooks.md` が担当する。
本ルールはそれ以外の `scripts/*.sh` と `Makefile` を対象とする。

## Makefile の書き方

- レシピは `@scripts/<name>.sh [args]` の 1 行呼び出しに限定する
  - 派生リポジトリで言語ネイティブのコマンドが 1 行で済むもの（`cargo build` 等）は直書きしてよい
  - 複数行・`for` / `if` などの条件分岐・環境依存の処理（`curl`・`docker` 等）は `scripts/` へ移す
  - 例外は `help` のみ（`## ` コメントから一覧を生成する 1 行の `awk`）
- 全ターゲットを `.PHONY` に列挙する。`.DEFAULT_GOAL := help` を維持する
- 各ターゲットに `## <説明>。直接: scripts/<name>.sh` 形式のコメントを付ける
  - `help` は正規表現 `^[a-zA-Z0-9_-]+:.*## ` に一致する行だけを表示する。`## ` が無いターゲットは一覧に出ない
  - ターゲット名の表示幅は `%-10s`（`test-hooks` でちょうど 10 文字）。11 文字以上の名前を足す場合は幅も広げる
- 中身の無いターゲット（`@true`・`echo` だけ等）は置かない。呼び出し元が終了コードを信用して素通りするため
- GNU Make 3.81（macOS 標準）互換を保つ。`.ONESHELL`・`.RECIPEPREFIX`・`define` + `$(eval)` の多用など 3.82+ 前提の機能は使わない
- Makefile 冒頭のコメント（役割・3.81 互換・help の仕組み）は削らない

## ターゲットを追加・改名・削除するときの手順

1. 処理の実体を `scripts/<name>.sh` に書く（下記「スクリプトの規約」）
2. `Makefile` に 1 行レシピ・`.PHONY`・`## ` コメントを追加する
3. `scripts/help.sh` の一覧を同じ変更で更新する（手動で保守する一覧であり、自動同期の仕組みは無い）
4. 呼び出し元を洗い出して更新する

    ```bash
    grep -rn "make \|scripts/" .github lefthook.yml README.md scripts .claude/rules
    ```

    - `ci.yml` の `make check` / `make test-hooks`、`lefthook.yml`、README の「コマンド」表・「派生リポジトリでの拡張」が主な呼び出し元
5. `make help` と `scripts/help.sh` の表示が一致すること、`make check` が通ることを確認する
6. 改名・削除は終了コードや引数の互換を壊す変更として PR 本文に明記する

## ターゲットの契約（本リポジトリに実在するもの）

| ターゲット | 実体 | 契約 |
| --- | --- | --- |
| `help` | Makefile の `awk`（Make 非依存の一覧は `scripts/help.sh`） | 一覧の表示のみ |
| `doctor` | `scripts/doctor.sh` | 読み取りのみ。何も導入・更新・修復しない。必須ツール欠落時は非 0 |
| `setup` | `scripts/setup.sh` | 冪等（再実行しても安全）。`lefthook install` と `.env` 雛形の配置のみで、既存の `.env` は上書きしない。ツール本体は導入しない |
| `check` | `scripts/check.sh` | ソースを変更しない（自動修正しない）。editorconfig-checker + shellcheck。CI の `check` ジョブと同一 |
| `test-hooks` | `scripts/hooks/test-hooks.sh` | 一時 git リポジトリで検証し作業ツリーを変更しない。依存は bash + git のみ。CI の `test-hooks` ジョブと同一 |

派生リポジトリでターゲットを足す場合の契約:

- `check` を拡張する場合も検査専用を保つ。自動修正は別ターゲット（例: `fix`）に分ける
- 生成物の削除（例: `clean`）はそのリポジトリが生成したディレクトリだけを対象にし、共有キャッシュ（`~/.cargo`・pnpm store 等）や追跡ファイルを消さない
- CI で呼ぶターゲットは、CI 側で独自のロジックを足さずに済む形にする（`.claude/rules/github-actions.md`）

## スクリプトの規約

- 先頭は `#!/usr/bin/env bash` と役割を説明する日本語コメント、続けて `set -euo pipefail`
- `scripts/lib.sh` を source し、その関数を使う

    ```bash
    DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
    # shellcheck source=./lib.sh
    source "${DIR}/lib.sh"
    ```

    - 引数を取らないスクリプトは `reject_args "$(basename "$0")" "$@"`（誤用は終了コード 2）
    - 外部コマンドは `require_cmd <name>` で確認する（導入は試みない。案内は `doctor.sh` / README の役割）
    - 進捗は `log`（stderr）。作業ディレクトリは `cd "${ROOT_DIR}"` で固定し、cwd に依存しない
- 実行時の出力（`log`・エラーメッセージ・usage）は英語、コード内コメントは日本語（既存スクリプトに合わせる）
- 実行権限を付ける（`chmod +x`。git 上で `100755` になっていること）
- shellcheck を通す（`.shellcheckrc` により source 先の `lib.sh` も追って検査される）。`make check` で確認する
- macOS 標準の bash 3.2 でも動く書き方にする（`mapfile`・連想配列・`${var,,}` は使わない）

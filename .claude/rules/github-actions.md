---
paths:
  - ".github/workflows/**"
description: >
  .github/workflows/ を編集する際の方針。runner 選択（public = GitHub ホステッド、ai-review の
  codex 専用 self-hosted runner は例外）、必須チェックと集約ジョブ ci-complete、発火条件・concurrency・
  paths フィルタの書き方、ジョブ追加時のチェックリスト、所要時間の計測方法と棚卸し結果を規定する。
---

# GitHub Actions 編集ルール

## 目的

CI は全 PR で走るため、ワークフロー数・ジョブ数・所要時間はそのまま待ち時間と課金分数になる。
**既存ワークフローへの集約・不要な発火の抑止・並列化**を優先し、ワークフロー・ジョブを安易に増やさない。
CI 独自のロジックは持たず、ローカルと同じ `make` / `scripts/` を呼ぶ（`.claude/rules/makefile.md`）。

## 現状のワークフロー

| ファイル | ジョブ（check 名） | トリガー | runs-on |
| --- | --- | --- | --- |
| `ci.yml` | `make check (...)` / `hook script tests (ubuntu-latest\|macos-latest)` / `PR title (...)` / `ci-complete` | `pull_request`・`push`（`main` のみ） | `ubuntu-latest`（`test-hooks` の matrix のみ `macos-latest`） |
| `ai-review.yml` | `codex / preflight`・`codex / review`・`codex / post_feedback`（Fandhe-AI/actions の reusable workflow） | `pull_request`（base が `main`。`opened` / `synchronize` / `reopened`） | preflight・post_feedback は `ubuntu-latest`、review は self-hosted の `codex` ラベル |
| `update-external.yml` | `update-external / *`（Fandhe-AI/actions の reusable workflow） | `schedule`（毎日 00:00 UTC）・`workflow_dispatch` | `ubuntu-latest`（`runner-json` で指定） |

## runner 方針

本リポジトリは **public** であり、組織方針（Fandhe-AI/actions `docs/runner-policy.md`）に従い
**GitHub ホステッド（`ubuntu-latest`）を使う**。private の派生リポジトリでは `self-hosted`（または絞り込んだラベル）を使う。

- public リポジトリで self-hosted runner を使うと、fork からの PR 経由で untrusted なコードが runner 上で
  実行され得る。新規ジョブで `runs-on: self-hosted` を書かない
- reusable workflow の runner 入力は既定値が可視性で自動切替しないものがある。public から呼ぶ場合は
  `ubuntu-latest` を明示する（`ai-review.yml` の `post-feedback-runner`・`update-external.yml` の `runner-json`）

### 例外 1: `test-hooks` の `macos-latest`

開発者の多くは macOS でフックを実行するため、BSD 系コマンドの挙動と macOS 標準の bash 3.2 を検証する目的で
`macos-latest` を使う（GitHub ホステッドのため runner 方針には反しない。分単価が高いので他ジョブには広げない）。

### 例外 2: `ai-review.yml` の review ジョブ（public で self-hosted）

`codex / review` は provider と同名のラベル `codex` を持つ self-hosted の codex 専用 runner で動く
（2026-10-11 の実行で runner `fandhe-ai-review-1-codex-runner-2`・ラベル `codex` を確認）。
組織方針 `docs/runner-policy.md` §3 が public リポジトリでも許容する例外と定めている。理由は、認証が runner 上の
ログイン済みディレクトリ（home-dir 方式）で、GitHub ホステッドでは資格情報を保持できないため。

fork PR で実行されない根拠（Fandhe-AI/actions の `latest` タグ = `187cd8b` を 2026-10-11 に確認した範囲）:

- reusable workflow `.github/workflows/ai-review.yml` の `preflight` ジョブは
  `if: github.event.pull_request.head.repo.full_name == github.repository` で fork PR では起動しない
- `review` は `needs: preflight` かつ `if: needs.preflight.outputs.run == 'true'` のため、preflight が起動しなければ実行されない
- `docs/runner-policy.md` §3 は例外の前提として「fork からの PR では実行しない」「ジョブユーザーに passwordless sudo を
  与えない」「信頼できないリポジトリへ runner を共有しない」を挙げ、review ジョブには sudo を検証する fail-closed の
  ステップがある

この保証は本リポジトリではなく上流（可変タグ `latest`）にある。上流の変更で前提が崩れていないかは、
`ai-review.yml` を変更するときに上記の条件を再確認する。`ai-review.yml` の trigger に `pull_request_target` を
使わない（fork PR に書き込み権限・secrets を渡すことになる）。

## 必須チェックと集約ジョブ

- **現状、必須チェックは未設定**（2026-10-11 時点で `gh api repos/Fandhe-AI/template-dev/rulesets` と
  `.../rules/branches/main` は空、`main` の branch protection も無い）
- README の想定では、ruleset の required status checks に `ci-complete` と `codex / preflight`・`codex / review`・
  `codex / post_feedback` を登録する（導入は `setup-repo-guards` スキル）。preflight が失敗すると後続は skipped になり、
  skipped は required check を満たすため、preflight も必ず登録する
- `ci.yml` のジョブを required に個別登録せず、集約ジョブ `ci-complete` だけを登録する
  - `ci-complete` は `if: always()` で全ジョブの結果を検査する fail-closed のジョブ。success 以外は失敗とし、
    skipped は明示的に許容したもの（`pr-title` の push イベント時）に限る
- ジョブ名（`name:`）・ジョブ ID・reusable workflow を呼ぶジョブ ID（`codex`）を変えると check 名が変わる。
  required に登録済みの名前を変える場合は ruleset 側も同時に更新する（更新漏れは「永久に pending」でマージ不能になる）

## 発火条件の方針

- `push` は default ブランチ（`main`）のみ。PR ブランチは `pull_request` で検査し、二重実行しない
- concurrency は ref 単位で、PR では古い実行を取り消す。`main` への push は取り消さない
  （連続マージで前のコミットの CI 結果が失われないようにする）

    ```yaml
    concurrency:
      group: ci-${{ github.workflow }}-${{ github.ref }}
      cancel-in-progress: ${{ github.ref != 'refs/heads/main' }}
    ```

    - reusable workflow のトップレベル concurrency は効かないため、呼び出し側（wrapper）で設定する
    - schedule・自動 PR 作成のように中断すると不整合が残るものは `cancel-in-progress: false`（`update-external.yml`）
- `paths` / `paths-ignore` は、required check を含むワークフロー（`ci.yml`・`ai-review.yml`）に
  **ワークフロー単位で付けない**。発火しないと check が報告されず pending のままマージ不能になる
  - ドキュメントのみの変更で重いジョブを省きたい場合は、ジョブ内の変更検知（`git diff` ベース等）で
    ジョブを skip し、`ci-complete` で「その条件での skipped」だけを許容する
  - 現状の `ci.yml` は全体で数十秒のため導入していない（費用対効果が見合う重いジョブを足した時に検討する）
- draft PR の skip は現状入れていない。入れる場合は `if: github.event.pull_request.draft == false` とし、
  trigger の `types` に `ready_for_review` を加える（加えないと ready にしても CI が走らない）
- schedule は日次以下の頻度にする（`update-external.yml` は 00:00 UTC の 1 回）

## ジョブ・ワークフローを追加するときのチェックリスト

1. 新規ワークフローを作らず、まず既存の `ci.yml` へのジョブ追加で済むか検討する（同じトリガーなら `ci.yml` に集約）
2. 追加したジョブを `ci-complete` の `needs` に必ず加える（漏れると未検査のままマージ可能になる）。条件付きで skip
    するジョブは、許容する条件を `ci-complete` の jq 式に明示する
3. 処理は `make <target>` / `scripts/*.sh` を呼ぶ。CI 側に複数行のロジックを書かない
    （ツール導入のように CI 固有のステップは例外）
4. lefthook と同じ検査を同じ範囲で重複させない。役割分担は `.claude/rules/git-hooks.md`（重い・網羅的な検査は CI が正）
5. 不要な `needs` で直列化しない。独立したジョブは並列に走らせ、クリティカルパス（最長ジョブ）を短くする
    - 逆に、セットアップ（checkout・ツール導入・キャッシュ復元）が実処理より長いほど細かく分けない
6. 依存のあるツールチェーンはキャッシュを使う（`actions/setup-node` の `cache`・`Swatinem/rust-cache` 等）。
    self-hosted で `CARGO_HOME` を共有する場合の rust-cache の注意は Fandhe-AI/actions `docs/runner-policy.md` §5
7. `actions/checkout` は既定の `fetch-depth: 1` のまま使い、履歴が必要な場合だけ深くする。submodule は必要時のみ取得する
8. 各ジョブに `timeout-minutes` と最小の `permissions` を付け、checkout は `persist-credentials: false`
9. ピン止め方針に合わせる
    - 公式・サードパーティ action は commit SHA で固定し、行末にタグをコメントする（`@<sha> # v7.0.1`）
    - 自組織の reusable workflow（`Fandhe-AI/actions`）は可変タグ `@latest`
    - CLI ツールはバージョン + SHA256 を `ci.yml` の `env` に宣言し、検証してから導入する（既存の `check` ジョブの方式）
10. 信頼できない入力（PR タイトル等）は `${{ }}` で `run` に直接展開せず `env` 経由で渡す（`pr-title` ジョブの方式）
11. 時間予算: `ci.yml` は PR の wall-clock で 2 分以内を目安とする。超える場合は並列化・キャッシュ・
    変更検知による skip を検討し、PR 本文に変更前後の実測を書く
12. 変更後は `actionlint` で検査し、PR 上で実際に CI を走らせて所要時間を確認する

## 所要時間の計測方法

```bash
# ワークフロー単位の実行時間（作成〜更新）。直近 30 件
gh run list --workflow ci.yml --limit 30 --json databaseId,conclusion,createdAt,updatedAt,event
# ジョブ単位の開始・終了
gh run view <run-id> --json jobs --jq '.jobs[] | [.name, .conclusion, .startedAt, .completedAt] | @tsv'
```

### 棚卸し結果（2026-10-11）

| ワークフロー | 件数 | 中央値 | 最大 | 主なジョブ（直近 1 件） |
| --- | --- | --- | --- | --- |
| `ci.yml` | 14 | 14 秒 | 69 秒 | check 約 7 秒・test-hooks ubuntu 約 6 秒 / macOS 約 20 秒・ci-complete 約 2 秒 |
| `ai-review.yml` | 7 | 50 秒 | 69 秒 | preflight 約 4 秒・review 約 37 秒・post_feedback 約 7 秒 |
| `update-external.yml` | 5 | 100 秒 | 106 秒 | skills 更新 約 96 秒・submodule 約 8 秒 |

この時点で push は `main` のみ・concurrency 設定済み・ジョブは並列で、ワークフローの統合や削除の余地は無かった。
変更点は `main` への push で `cancel-in-progress` を無効化したことのみ。

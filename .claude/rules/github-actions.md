---
paths:
  - ".github/workflows/**"
description: >
  .github/workflows/ を編集する際の方針。runner 選択（public = GitHub ホステッド、ai-review の
  codex 専用 self-hosted runner は例外）、必須チェックと集約ジョブ ci-complete、発火条件・concurrency・
  paths フィルタの書き方、ジョブ追加時のチェックリスト、キャッシュ方針（容量 10 GB・保存は main の push のみ・
  キー設計・効果の計測）、所要時間の計測方法と棚卸し結果を規定する。
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
6. キャッシュは効果を測ってから入れる。保存は `main` への push に限り、キーはルートの lockfile のハッシュで作る
    （「キャッシュ方針」節）。self-hosted で `CARGO_HOME` を共有する場合の rust-cache の注意は
    Fandhe-AI/actions `docs/runner-policy.md` §5
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

## キャッシュ方針

GitHub Actions のキャッシュ（`actions/cache`・`actions/setup-node` の `cache`・`Swatinem/rust-cache` 等）を
追加・変更するときの方針。効かないキャッシュは復元時間と容量を浪費し、他のキャッシュを追い出すため、
「入れれば速くなる」とは考えない。

### 現状（2026-10-11 時点）

- 本リポジトリは GitHub Actions のキャッシュを使っていない（キャッシュ 0 件・0 バイト）。
  `ci.yml` のツールは `env` に宣言した版を SHA256 で検証して毎回導入する方式で、所要時間は数十秒
- 言語固有のジョブを足してキャッシュを入れる場合は、本節に従う

### 前提: 容量と課金

- リポジトリあたりの上限は 10 GB（Org 設定も 10 GB）。超えると最終アクセスの古い順に追い出され、
  7 日間アクセスの無いキャッシュは削除される
- Org の Actions 予算は $0 で、超過時は停止する。そのため 10 GB を超える上限は使えない
  （上限を上げると、10 GB を超えた時点でキャッシュが読み取り専用になる）。上限・予算の変更はユーザーが判断する

### 入れる前に効果を測る

- self-hosted runner ではツールチェーン・pnpm store・`CARGO_TARGET_DIR` 等がジョブをまたいで残ることがあり、
  GitHub のキャッシュが無意味（復元時間だけかかる）な場合がある。本リポジトリは GitHub ホステッドだが、
  private の派生リポジトリで self-hosted を使う場合は特に確かめる
- 組織内の実例
  - mirror-ui: pnpm キャッシュの復元に 1 ジョブ平均 75 秒かけていたが、install は `downloaded 0` で効果が無かったため撤去した
  - team-hub: `Swatinem/rust-cache` が `./target` を保存対象にしていたが、runner の `CARGO_TARGET_DIR` が別の場所で
    registry しか保存されておらず、さらに後処理が共有 `CARGO_HOME` の crate を消していたため撤去した
- キャッシュを足す・残す前に、キャッシュ有無のジョブログで効果を確かめる。見るのはパッケージの `downloaded` 件数、
  依存の再ビルド数（`Compiling` の行数）、復元にかかった秒数

### 保存は main への push だけ

PR から保存したキャッシュは `refs/pull/<N>/merge` スコープに入り、その PR からしか読めないのに容量を共有し、
`main` のキャッシュを追い出す。保存は `main` への push に限る。

- `actions/cache` は restore（全イベント）と save（`main` の push のみ）に分ける

    ```yaml
    - name: Restore cache
      id: cache
      uses: actions/cache/restore@<sha> # vX.Y.Z
      with:
        path: <path>
        key: <key>
        restore-keys: <prefix>
    # ...ビルド・テスト...
    - name: Save cache
      if: github.event_name == 'push' && github.ref == 'refs/heads/main' && steps.cache.outputs.cache-hit != 'true'
      uses: actions/cache/save@<sha> # vX.Y.Z
      with:
        path: <path>  # restore と同じ
        key: ${{ steps.cache.outputs.cache-primary-key }}
    ```

- `Swatinem/rust-cache` は `save-if: ${{ github.event_name == 'push' && github.ref == 'refs/heads/main' }}` を付ける
  （`github.ref` だけだと `main` 上の `schedule`・`workflow_dispatch` でも保存する。下の `cache-save` も同じ）
- Fandhe-AI/actions の `rust-base-ci.yml` を `cache: true` で呼ぶ場合は `cache-save: ${{ github.event_name == 'push' && github.ref == 'refs/heads/main' }}`
  を渡す（`cache` の既定は false、`cache-save` の既定は true のため、渡さないと PR でも保存する）
- `actions/setup-node` の `cache: pnpm` のように restore と save が一体の action は、PR でも保存する点に注意する

### キーの設計

- キーには中身のハッシュ（lockfile 等）を入れる。`github.sha`・`github.run_id` など毎回変わる値は入れない
  （入れる場合は restore-keys の前方一致で直近を拾う設計にし、保存は `main` に限る）
- `hashFiles('**/Cargo.lock')`・`hashFiles('**/pnpm-lock.yaml')` のような再帰 glob は使わず、ビルド対象に本当に入る
  ルートの lockfile を指定する。再帰 glob では `.agents/skills/*/samples/**` などスキルのサンプルの lockfile が混入し、
  依存が変わらないのにキーが変わって全キャッシュを作り直す（fandhe-browser・fandhe-frontend・fandhe-db・
  fandhe-container・template-frontend-react_router で発生）
- ビルド成果物（Rust の `target/` 等）をキャッシュするなら、キーにツールチェーンの版を入れる（`rustc -vV` の
  release・commit-hash、`dtolnay/rust-toolchain` の `cachekey` 出力等）。入れないと runner イメージの更新後も古い成果物に
  exact hit し続け、exact hit では再保存されないため依存を毎回全再ビルドする（fandhe-db で依存 135 crate を毎回再ビルドしていた）
- restore-keys で前方一致の部分復元を許し、キーが変わっても cold にならないようにする
- キーが捕捉しない変更で保存内容が変わるときは、キーの版セグメント（`-v2-` 等）を上げる。restore-keys の版も同時に上げ、
  版セグメントが無ければ追加する。対象はルートの `Cargo.toml` の `[profile]`、`RUSTFLAGS`・`CARGO_PROFILE_*` 等の env、
  `.cargo/config.toml`、ビルドコマンド・フラグなど（Rust の具体例は `.claude/rules/rust-build-profile.md`）。上げないと
  ツールチェーンの版と同じく古い成果物に exact hit し続ける。キーが `Cargo.toml` を含んで変わる場合も、restore-keys で
  旧世代を部分復元すると新旧の依存成果物が混ざった大きな初回保存になるため、版を上げて旧世代を拾わないようにする
- `Swatinem/rust-cache` はキーに `CARGO`・`RUST` 等で始まる env（action を実行した時点の値）・`.cargo/config.toml`・
  `rust-toolchain(.toml)`・**workspace メンバーの** `Cargo.toml` を含める（上流 `src/config.ts` は `cargo metadata --no-deps`
  の packages の manifest だけをハッシュする）。ルートが仮想 manifest（`[package]` なし）だとルートの `Cargo.toml` は
  ハッシュされず、ルートの `[profile]` の変更はキーに入らない。これや、後続のステップで設定する env・ビルドコマンドの
  フラグを変えたときは、`prefix-key` の版（既定 `v0-rust`）を上げる（fandhe-edge-model #530）
- Fandhe-AI/actions の `rust-base-ci.yml` を `cache: true` で呼ぶ場合、clippy・test のキーは
  `<cache-key-prefix>-<OS>-{clippy,test}-v4-<rustc の版>-<Cargo.lock のハッシュ>`（`cache-key-prefix` の既定は `rust-base-ci`）で、
  版セグメントは呼び出し側から変えられない（`cache-key-prefix` を変えると deny ジョブを含む全キャッシュが cold になる）。
  キーに入らない変更をしたら、**マージ後に** `main` スコープの `<prefix>-<OS>-{clippy,test}-v4-*` を削除する。
  マージ前に消すと、その間の `main` への push が旧設定の内容を同じキーで保存し直す

    ```bash
    # clippy・test それぞれ、OS（runner.os: Linux 等）ごとに一覧して id で消す
    gh cache list -L 100 --ref refs/heads/main --key <prefix>-Linux-clippy-v4- --json id,key
    gh cache delete <id>
    ```

### サイズを抑える

- 1 世代（キャッシュ種別ごとの最新 1 件）の合計が 10 GB に余裕をもって収まるようにする。
  キャッシュ種別を増やすときは「1 件のサイズ × 種別（ジョブ）× OS」を見積もる
- Rust なら保存前に workspace メンバーの成果物を prune する、`CARGO_INCREMENTAL=0`、CI 限定で
  `CARGO_PROFILE_DEV_DEBUG=line-tables-only` 等で縮める

### 追い出されても hit する設計

- 上限に達しても、毎回読まれる `main` の最新キャッシュは最後まで残る（最終アクセスの古い順に追い出されるため）。
  キーの切り替えで古い世代が追い出されても、新しい世代が保存済みならミスにならない
- 保存を `main` に限り、キーが不要に変わらない状態を保てていれば上限を上げる必要はない

### 計測コマンド

```bash
# 使用量（件数・バイト数）
gh api repos/{owner}/{repo}/actions/cache/usage
# 一覧（key・サイズ・ref・最終アクセス）
gh cache list -L 100
# 不要なキャッシュの削除（共有状態のため、key と ref を確認してから消す）
gh cache delete <id>
```

ジョブログでは次の行を見る。

- `Cache restored from key: <key>`: primary key と一致すれば exact hit、それ以外は restore-keys による部分復元
- `Cache not found for input keys: ...`: ミス
- `Cache saved with key: <key>`: 保存された（PR で出ていれば保存条件の誤り）

### キャッシュを追加・変更する PR のチェックリスト

PR 本文に次を確認したことを書く。

1. 効果を測った（キャッシュ有無の `downloaded` 件数・再ビルド数・復元秒数）
2. 保存が `main` への push に限られている
3. キーが中身のハッシュ（ルートの lockfile のみ）と、成果物を含むならツールチェーンの版でできており、restore-keys がある
4. 1 世代の合計サイズを見積もり、10 GB に余裕をもって収まる

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

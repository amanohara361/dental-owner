# 運用メモ（定時実行の仕組みと既知の制約）

最終更新: 2026-08-07

## 定時実行の構成

3本の Routine（スケジュール実行）が登録されている。各回は**新しいセッション**が
起動し、リポジトリを取得してから `prompts/` 配下の手順書に従って作業する。

| トラック | 実行 (JST) | cron (UTC) | 手順書 |
| --- | --- | --- | --- |
| 法令・手続き・人事 | 月 17:00 | `0 8 * * 1` | `prompts/regulatory.md` |
| 財務 | 火 08:00 | `0 23 * * 1` | `prompts/weekly-finance.md` |
| 立地 | 木 13:00 | `0 4 * * 4` | `prompts/weekly-location.md` |

## 既知の制約1: セッションにリポジトリが自動で渡らない

**2026-08-07 に判明。初回の試走が失敗した原因。**

Routine から起動したセッションは、作業ディレクトリにリポジトリが
チェックアウトされていない状態で立ち上がる
（`no git repository found in /home/user`）。手順書もチェックリストも
読めないため、そのままでは作業が成立しない。

### 対処

各 Routine のプロンプト冒頭に、リポジトリを取得する手順を入れてある。
**Routine を新しく作る／作り直すときは、この手順を必ず先頭に置くこと。**

```
作業前の準備。リポジトリが未取得の状態で起動する場合があるため、まず次を実行する。
1. /home/user/dental-owner が無ければ
   cd /home/user && git clone https://github.com/amanohara361/dental-owner
2. cd /home/user/dental-owner
3. git fetch origin claude/dental-startup-advisor-ctrpar
4. git checkout claude/dental-startup-advisor-ctrpar
5. git pull origin claude/dental-startup-advisor-ctrpar
これが終わってから本題に入る。
```

検証結果（2026-08-07）:

| 方式 | 結果 |
| --- | --- |
| リポジトリ指定なし | 失敗。git リポジトリが存在せず着手不能 |
| セッション作成時に `source_url` を指定 | 成功。commit → push まで通る |
| プロンプト内で自力 `git clone` | **成功**。Routine ではこの方式を採用 |

push の権限自体には問題がない。

## 既知の制約2: 一部サイトに直接アクセスできない

実行環境のネットワーク制限により、以下は `WebFetch` でブロックされる
（`EGRESS_BLOCKED`）。

- `www.jfc.go.jp`（日本政策金融公庫）
- `www.mhlw.go.jp`（厚生労働省）

Web検索そのものは通るため、**検索結果から一次情報の内容を拾うことはできるが、
官公庁ページを直接開いて原文を確認することはできない**。

### 対処

- 一次情報を直接確認できなかった項目は、レポートに**「未確認・二次情報」と明記**する。
  数字を確定情報として書かない
- 融資条件・施設基準・届出要件など、外すと損害が出る項目は
  「窓口で要確認」を必ず添える
- 制限を外したい場合は、環境のネットワーク設定で許可ドメインを追加する必要がある
  （claude.ai の環境設定から。**未対応**）

## トラブル時の切り分け手順

レポートが出ていないときは、次の順で確認する。

1. リモートブランチに新しいコミットがあるか
   （`git ls-remote origin refs/heads/claude/dental-startup-advisor-ctrpar`）
2. Routine が有効か、`next_run_at` が想定どおりか
3. 該当セッションの終了状態。`status_category` が `need_input` なら
   何かで詰まって止まっている
4. 切り分けには、調査をさせず git 操作だけを実行させる
   **最小プローブセッション**を投げるのが速い

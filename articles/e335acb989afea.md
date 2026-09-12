---
title: "Rust製エディタ「綴」開発ログ：ワークスペース切り替え・未保存保護・セッション復元まで実装した"
emoji: "⛳"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["rust", "綴エディタ", "開発記録"]
published: false
---
# Rust製エディタ「綴」開発ログ：ワークスペース切り替え・未保存保護・セッション復元まで実装した

個人制作しているRust製エディタ「綴（Tsuzuri Editor）」の開発を進めた。



今回の大きな目標は、単なるテキストエディタから一歩進めて、

**複数の開発プロジェクトを切り替えながら作業できるワークスペース型エディタ**

にすること。

最終的に、

```text
ワークスペース登録
    ↓
別プロジェクトを選択
    ↓
未保存ファイルを自動保存
    ↓
現在のセッションを保存
    ↓
対象ワークスペースで綴を再起動
    ↓
前回開いていたタブ・UI状態を復元
```

という一連の流れが動くところまで完成した。

releaseビルドもエラーなし。

今回は、その実装過程をまとめる。

---

## 綴とは

「綴」は、自分の開発作業向けに作っているRust製の軽量エディタ。

主に、

- Rust
- `.8g`という自作スクリプト
- JSON
- Python
- Markdown
- Blenderアドオン

などを編集する用途を想定している。

UIはegui / eframeベース。

Zedのように、

「プロジェクトを開いて、その中でコード・Git・Task・ログをまとめて扱う」

という方向を目指している。

最近は文字サイズ設定やJSON設定ファイルの分離、シンタックスハイライト、Explorer、Git表示、Tasks、Emmetなども少しずつ実装している。

今回追加したのは、その中でもかなり重要な**ワークスペース機能**。

---

# Phase 7：ワークスペース一覧を作る

まず実装したのが、ワークスペースの登録機能。

左上に小さなワークスペースパネルを配置し、

```text
綴
燕
稲荷
八百よろず
```

のように複数の開発フォルダを登録できるようにした。

内部では、

```text
%APPDATA%\Tsuzuri\workspaces.json
```

へワークスペース情報を保存する。

データ構造はこんな感じ。

```rust
#[derive(Clone, Debug, PartialEq, Eq, Serialize, Deserialize)]
pub struct WorkspaceEntry {
    pub name: String,
    pub folder: PathBuf,
    pub last_opened: String,
}
```

そして台帳全体。

```rust
#[derive(Clone, Debug, PartialEq, Eq, Serialize, Deserialize)]
pub struct WorkspaceRegistry {
    pub version: u32,
    pub workspaces: Vec<WorkspaceEntry>,
}
```

これで、

```text
名前
フォルダ
最終使用日時
```

を保持できる。

一覧では最近開いた順に並べるようにした。

---

## Rustのモジュール衝突に遭遇

途中でこんなエラーが発生した。

```text
failed to resolve mod `workspace`:
file for module found at both

src\workspace.rs
src\workspace\mod.rs
```

Rustでは、

```text
src/workspace.rs
```

と

```text
src/workspace/mod.rs
```

は両方とも、

```rust
mod workspace;
```

の本体候補になる。

つまり、両方存在すると衝突する。

もともと存在していた `workspace/mod.rs` に加えて、ワークスペース台帳用として `workspace.rs` を追加してしまったのが原因だった。

最終的には、

```text
src/
└─ workspace/
   ├─ mod.rs
   ├─ registry.rs
   └─ switching.rs
```

という構成に整理した。

`registry.rs` にワークスペース台帳処理を分離。

`mod.rs` 側では、

```rust
mod registry;

pub use registry::*;
```

として公開する形にした。

個人的にもこの構造のほうが好き。

今後機能が増えても、

```text
workspace/
├─ registry.rs
├─ switching.rs
├─ config.rs
└─ session.rs
```

みたいに整理できる。

---

# Windowsパスの罠

その後、テスト94件中1件だけ失敗した。

```text
workspace::registry::tests::一覧解除しても実フォルダは削除しない
```

エラーは、

```text
ワークスペースが見つかりません
```

だった。

原因はWindowsのパス。

登録時には、

```rust
fs::canonicalize(folder)
```

を使っていた。

Windowsでは `canonicalize()` を通すと、

```text
\\?\E:\project
```

のような形式になることがある。

一方、削除時には、

```text
E:\project
```

がそのまま渡されていた。

元々の比較処理は、

```rust
fn workspace_key(path: &Path) -> String {
    path.to_string_lossy()
        .replace('/', "\\")
        .to_lowercase()
}
```

だったため、

```text
\\?\E:\project
```

と

```text
E:\project
```

が別物として判定されてしまう。

そこで比較時にも正規化するよう変更。

```rust
fn workspace_key(path: &Path) -> String {
    // Windowsでは canonicalize 後に "\\?\" 付きのパスになることがあるため、
    // 登録時と検索・削除時で同じ形式へ正規化してから比較する。
    let normalized =
        fs::canonicalize(path).unwrap_or_else(|_| path.to_path_buf());

    normalized
        .to_string_lossy()
        .replace('/', "\\")
        .to_lowercase()
}
```

これでテストは通過した。

Windowsアプリを作っていると、パス周辺はやっぱり要注意。

---

# Phase 8：本当にワークスペースを切り替える

Phase 7では一覧表示まで。

Phase 8では、

**選択したワークスペースへ実際に移動する**

機能を追加した。

ここで一番重要にしたのが、

**編集中の内容を絶対に失わないこと。**

仕様はこうした。

```text
ワークスペース選択
        ↓
未保存タブ確認
        ↓
未保存あり
        ↓
自動保存
        ↓
保存成功
        ↓
セッション保存
        ↓
新ワークスペースで綴を起動
        ↓
旧ウィンドウを閉じる
```

保存に失敗した場合は切り替えない。

新しい綴の起動に失敗した場合も、現在の綴は閉じない。

「保存せずに切り替える」という経路は作らなかった。

---

## `--workspace` で対象フォルダを渡す

ワークスペース切り替えは、同じプロセスの中で全部を差し替える方式ではなく、

**新しい綴プロセスを起動する**

方式にした。

例えば、

```powershell
tsuzuri.exe --workspace E:\tsubame-stream
```

のように起動する。

引数生成はシンプル。

```rust
pub fn workspace_process_args(folder: &Path) -> [OsString; 2] {
    [
        OsString::from("--workspace"),
        folder.as_os_str().to_owned(),
    ]
}
```

起動側では、

```rust
std::process::Command::new(executable)
    .args(args)
    .current_dir(folder)
    .spawn()
```

で新しいプロセスを立ち上げる。

新しいプロセスが起動できたときだけ、

```rust
egui::ViewportCommand::Close
```

を現在ウィンドウへ送る。

この順番はかなり重要。

先に閉じてしまうと、新プロセス起動失敗時に作業中の綴まで消えてしまう。

---

# 未保存ファイルは自動保存

当初は、

```text
すべて保存して切り替え
各ファイルを確認
キャンセル
```

のような確認ダイアログも検討した。

ただ、自分用エディタとして使うなら毎回確認が出るのは少し面倒。

そこで最終的には、

**未保存なら自動保存してから切り替える**

仕様にした。

まだ保存先が存在しない新規ファイルだけは、保存ダイアログを出す。

保存先選択をキャンセルした場合は、ワークスペース切り替えそのものを中止する。

これで操作数を増やさず、安全性も保てる。

---

# Phase 9：ワークスペースごとの状態復元

次に確認したのが、ワークスペースを切り替えたあと、

**前に何を開いていたか覚えているか**

という部分。

実はこの機能はすでに既存のセッション機能としてかなり実装されていた。

セッションは、

```text
<workspace>/.tsuzuri/session.json
```

へ保存される。

復元対象は、

```text
開いていたファイル
アクティブなタブ
最近使ったファイル
Explorer / Tasks / Git の選択
下部パネル
Explorer展開状態
サイドバー幅
下部パネル高さ
ウィンドウ位置
ウィンドウサイズ
最大化状態
```

など。

かなり残っていた。

復元処理では、

```rust
for relative in &session.open_files {
    let file = self.project_root.join(relative);

    if file.is_file() {
        self.open_file(file);
    }
}
```

としている。

つまり、以前開いていたファイルが削除されていても、

**存在するファイルだけ復元する。**

セッションに古いパスが残っていても、それだけで起動不能にはならない。

---

## 実際に往復してみる

実機では、

```text
綴
↓
燕
↓
綴
```

とワークスペースを往復。

綴側で開いていたファイルを残した状態で燕へ移動し、再び綴へ戻った。

結果、

**ちゃんと前回のタブとUI状態が復元された。**

ここはかなり嬉しかった。

エディタでプロジェクトを切り替えたとき、

「毎回ファイルを開き直す」

のは地味に面倒。

前回の作業状態へ戻れるだけで、かなり実用品っぽくなる。

---

# Git worktreeも使った

今回の開発では、

```text
E:\tsuzuri-editor
```

本体とは別に、

```text
E:\tsuzuri-worktrees\workspace-switching
```

というGit worktreeを作って作業していた。

featureブランチは、

```text
feature/workspace-switching
```

。

Phase 8完了後、

```powershell
git commit -m "feat: add safe workspace switching"
```

してから本体へマージ。

最後に、

```powershell
git worktree remove "E:\tsuzuri-worktrees\workspace-switching"

git branch -d feature/workspace-switching
```

でworktreeを削除。

最終的には、

```text
E:\tsuzuri-editor
```

へ全部統合した。

実装途中の壊れた状態を本体から隔離できるので、worktree方式はかなり便利だった。

---

# Git作者情報を設定していなくてコミットできなかった

途中、

```text
Author identity unknown
```

というエラーにも遭遇。

Gitの、

```text
user.name
user.email
```

が設定されていなかった。

そこで、

```powershell
git config user.name "名前"
git config user.email "メールアドレス"
```

を設定。

その後、

```powershell
git commit -m "feat: add safe workspace switching"
```

で無事コミットできた。

コードではなくGit環境側の問題だった。

---

# Phase 9パッチのREDテストでハマる

Phase 9では、欠損ファイル復元用テストを追加しようとした。

ところが、

```text
running 0 tests
98 filtered out
```

となった。

最初はCargo側の問題かと思ったが、実際には、

**指定したテスト名が現在の `session.rs` に存在していなかった。**

`cargo test -- --list`

で確認すると、認識されていたセッションテストは、

```text
old_v1_session_gets_safe_layout_defaults
session_paths_are_saved_relative_to_project_root
unknown_saved_tabs_fall_back_to_safe_defaults
workspace_session_json_round_trip_preserves_state
```

など。

つまりテストランナーは正常。

単純に、こちらが想定していたREDテスト名と現在のコードが一致していなかった。

最終的には現在の `restore_workspace_session()` を確認したところ、必要な復元処理はすでに実装済みだった。

そのため、余計なコードは追加せず、実機検証だけ行った。

---

# Releaseビルド

最後にreleaseビルド。

```powershell
cargo build --release
```

実行。

**エラーなし。**

生成された実行ファイルは、

```text
target\release\tsuzuri_task_runner_patch.exe
```

。

release版でも起動確認。

ここまでで、

```text
ワークスペース登録
ワークスペース検索
別ワークスペース選択
未保存ファイル自動保存
安全なプロセス切り替え
セッション保存
タブ復元
UI状態復元
```

まで一通り繋がった。

---

# 現在の綴

今回の開発で、綴はかなり「普段使いできるエディタ」に近づいた。

以前は、

```text
コードを書く
Taskを見る
Gitを見る
```

という単体機能が中心だった。

今は、

```text
綴を起動
↓
作業するプロジェクトを選択
↓
コード編集
↓
別プロジェクトへ切り替え
↓
戻る
↓
前の作業状態がそのまま復元
```

という開発フローが成立している。

個人的にはこの差は大きい。

エディタというより、

**自分の開発環境そのもの**

に少しずつ近づいてきた。

---

# 次は「使いながら直す」

ここからしばらくは、新機能を大量に増やすより、

**実際の開発で綴を使ってみる**

段階に入ろうと思う。

燕の開発、稲荷ブラウザ、八百よろずエンジン、Blenderアドオンなどを綴で編集して、

```text
ここが面倒
ここにショートカットが欲しい
この表示が狭い
この操作を自動化したい
```

という部分を拾っていく。

作っている最中には気づかない不便も、普段使いすると一瞬で見つかる。

なので次のPhaseは、使ってから決める予定。

---

## まとめ

今回かなり大きかったのは、

**「ファイルを編集できるエディタ」から「作業場所を覚えているエディタ」へ進んだこと。**

ワークスペースを切り替えて、戻ってきたら前の状態が残っている。

それだけでも開発体験はかなり変わる。

Rust + eguiでエディタを自作するのは地味な作業も多いけれど、自分の使い方に合わせて少しずつ育っていくのは面白い。

ひとまず今回の区切りとして、

**綴はreleaseビルドまで成功。**

ここから実戦投入してみる。

次は実際に普段使いして見つかった改善点を書いていく予定。

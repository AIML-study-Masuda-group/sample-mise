# mise のサンプルリポジトリ

（注：mise の仕様は将来変わる可能性があるので、公式サイトの最新版も参照したほうが安全です）

---

## 目次

1. mise とは
2. mise のインストール方法
3. グローバル設定（ユーザ環境）
4. プロジェクト毎設定（`mise.toml`）
5. uv（Python 依存管理ツール）との連携
6. GitHub Actions サンプル（プロジェクト毎運用向け）
7. 補足・注意点

---

## 1. mise とは

* mise（mise-en-place）は、**多言語/多ツールのバージョン管理／切り替え**、環境変数の切り替え、タスクランナー機能を統合で持つツールです。([Mise-en-place][1])
* `asdf` プラグイン互換性があり、既存の `.tool-versions` などもある程度読めます。([Mise-en-place][2])
* プラグインを通じて、Python、Terraform、Node.js、Ruby、Go などを扱えます。([Mise-en-place][2])
* Python に関しては、mise は uv（または pipx 相当の機能を持つバックエンド）と統合できるよう設計されています。([Mise-en-place][3])

---

## 2. mise のインストール方法

以下は主要な OS 向け手順。最新版は [Installing Mise (公式サイト)](https://mise.jdx.dev/installing-mise.html) を参照してください。([Mise-en-place][4])

### Linux / macOS での設定になります。 Windows は未調査。

```sh
curl https://mise.run | sh
```

このコマンドは mise のインストール用スクリプトを取得して実行します。([Mise-en-place][5])

一般的には、`~/.local/bin` 等に mise バイナリが配置されます（インストール先は環境変数で変更可能）([Mise-en-place][5])

#### シェルへの統合（例：bash, zsh）

インストール後、シェル起動時に mise を有効化するよう設定を追加します。公式ドキュメントに各シェル向け例があります。([Mise-en-place][4])

例：bash／zsh の場合

```sh
echo 'eval "$(mise activate bash)"' >> ~/.bashrc
# あるいは
echo 'eval "$(mise activate zsh)"' >> ~/.zshrc
```

その後、シェルを再起動または `source ~/.bashrc` 等で反映。([Mise-en-place][4])

補足：オートコンプリートスクリプトも `mise completion` コマンドで生成できます。([Mise-en-place][4])

### Homebrew / パッケージマネージャ経由（macOS / Linux）

⚠️以下の方法は今回は使わない予定です。

* Homebrew を使える環境なら：

  ```sh
  brew install mise gnupg
  ```

  などの方法もサポートされています。([stuartellis.name][6])

* 他 OS やパッケージマネージャでも、mise が提供されていれば同様にインストール可能です。([stuartellis.name][6])

### アンインストール

* `mise implode` コマンドで mise およびそのデータを一括削除できます。([Mise-en-place][4])
* 手動で以下ディレクトリを削除してもよい（`~/.local/share/mise`、`~/.config/mise`、`~/.cache/mise` 等）([Mise-en-place][4])

---

## 3. グローバル設定（ユーザ環境）

mise はユーザレベルでの **グローバルなデフォルト設定** を持たせることができます。たとえば、全体として使いたい Python や Terraform のバージョンをここで指定しておき、プロジェクト毎設定がなければこれが使われるようにする、という構成です。

* `~/.config/mise/config.toml` 等にグローバルの `[tools]` セクションを定義。[サンプルはこちら](/mise/config.toml)。実際はローカルユーザーの適切な位置に置いてください。

* たとえば：

  ```toml
  [tools]
  python = "3.12"
  terraform = "1.8.5"
  uv = "latest"
  ```

* この設定は、プロジェクト内に `mise.toml` がないときに適用されます。

つまり、ローカルで何も指定しなければユーザ全体で定義しておいたバージョンが使われるというベースラインを作れます。

---

## 4. プロジェクト毎設定（mise.toml）

プロジェクト（リポジトリ）直下に `mise.toml` を置き、その中で使いたいツールとバージョン、タスク定義などを記述します。

### 基本構成例

参考: [mise.toml](/mise.toml)

```toml
min_version = "2025.9.30"

[tools]
python    = "3.11.9"
terraform = "1.7.4"
uv        = "0.4.20"

[env]
_.python.venv = { path = ".venv", create = true }

[tasks]
"py:sync"  = "uv sync --frozen"
"py:test"  = "uv run pytest -q"
"tf:init"  = "terraform -chdir=infra init -upgrade"
"tf:plan"  = "terraform -chdir=infra plan"
"tf:apply" = "terraform -chdir=infra apply -auto-approve"
"tf:fmt"   = "terraform -chdir=infra fmt -recursive"
```

解説：

* `min_version`：このプロジェクトで必要とされる最低の mise バージョン
* `[tools]`：プロジェクトで使うツールとバージョンを指定
* `[env]`：環境変数や venv の自動作成設定など
* `[tasks]`：そのプロジェクトでよく使うコマンドをミニタスクにまとめておく

mise は、ディレクトリ移動時にそのディレクトリか親ディレクトリを見て `mise.toml` を探し、そこで定義されたツールバージョンに自動で切り替えます。([Mise-en-place][2])

また、`mise install` を実行すると、この `mise.toml` で定義されたツールをインストールします。([Mise-en-place][7])

### 利用可能なバージョンの確認

`mise.toml` に書くバージョン番号を決める際は、`mise ls-remote <tool>` でリモートに公開されている候補を一覧できます。([Mise-en-place][9])

```bash
# 例: Python の候補を一覧
mise ls-remote python

# 例: 3.11 系だけ見たい場合
mise ls-remote python 3.11
```

`mise plugins install <tool>` でプラグインを追加した直後でも、同コマンドで候補が確認できます。パッケージごとの最新安定版だけを知りたい場合は `mise ls-remote <tool> --latest` を使います。

インストール済み／現在有効なバージョンを確認したいときは次のコマンドが便利です：

```bash
# ツールごとにインストール済みバージョンを表示
mise ls <tool>

# カレントディレクトリで有効になっているバージョンを表示
mise current
```

これらを組み合わせることで、`mise.toml` に記載するバージョンを検討する際の材料が揃います。

---

## 5. uv（Python 依存管理ツール）との連携

Python 環境を扱う際に、Python 本体のバージョンは `mise` で管理し、**依存関係 / 仮想環境管理は `uv`** を使う形が便利です。

* mise の Python プラグインは uv をバックエンドとして扱えるよう実装されており、Python と uv の統合が可能です。([Mise-en-place][3])
* `.python-version` ファイルをサポートしつつ、mise のプラグインで Python を切り替え、uv で仮想環境を自動生成／同期できます。([Mise-en-place][3])
* GitHub や issue 等で、mise と uv の間でバージョン優先順位や pipx／uv 関連の挙動について議論されており、若干の注意が必要という報告もあります。([GitHub][8])

1. [pyproject.toml](/pyproject.toml)を作成します(`uv init`でもOK)。
参考 `uv add requests`, `uv add pytest --dev`
2. `uv lock`を実行します。
3. `uv sync ---frozen`を実行します(`mise run py:sync`でもOK)。
4. `uv run python app/main.py`で実行。[Python内の参考](/app/main.py)。

### 運用例

1. `mise install` → Python（3.11.x など）をインストール。プロジェクト外だと、グローバルで設定したものが、プロジェクト内だと`mise.toml`が優先されます。

  ローカルのグローバル環境の例

  ```bash
  % python --version
  Python 3.12.11
  % terraform --version
  Terraform v1.8.5
  on darwin_arm64

  Your version of Terraform is out of date! The latest version
  is 1.13.3. You can update by downloading from https://www.terraform.io/downloads.html
  % uv --version
  uv 0.8.22 (ade2bdbd2 2025-09-23)
  ```

  プロジェクト内の例

  ```bash
  % python --version
  Python 3.11.9
  % terraform --version
  Terraform v1.7.4
  on darwin_arm64

  Your version of Terraform is out of date! The latest version
  is 1.13.3. You can update by downloading from https://www.terraform.io/downloads.html
  % uv --version
  uv 0.4.20 (0e1b25a53 2024-10-08)
  ```

2. プロジェクトで `uv sync --frozen` を実行 → `uv.lock` に基づいてパッケージを厳密にインストール
3. タスク定義で `uv run pytest` や `uv run myscript` のように `uv run` 経由でコマンドを実行

このように、Python 本体まわりの切り替えとパッケージ管理を明確に分けられる構成が理想です。

---

## 6. GitHub Actions サンプル（プロジェクト毎運用前提）

以下は、プロジェクトリポジトリに `mise.toml` を置いている前提で、GitHub Actions 上でその通りの環境を再現し、テストや Terraform 操作を回す例です(未検証)。

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Install mise
        run: |
          curl https://mise.run | sh
          echo "$HOME/.local/bin" >> $GITHUB_PATH

      - name: Install tools defined in mise.toml
        run: mise install

      - name: Setup Python dependencies
        run: mise run py:sync

      - name: Lint / Test
        run: |
          mise run py:test

      - name: Terraform plan
        run: |
          mise run tf:init
          mise run tf:fmt
          mise run tf:plan

      - name: Cache setup (任意)
        uses: actions/cache@v4
        with:
          path: |
            ~/.cache/mise
            .venv
            uv.lock
          key: ${{ runner.os }}-mise-${{ hashFiles('mise.toml') }}-uv-${{ hashFiles('uv.lock') }}
```

ポイント：

* `mise install` によって、リポジトリの `mise.toml` に書いてあるツールをすべてインストール
* `mise run <task>` 形式で、ローカルで使っているタスクをそのまま CI でも実行
* キャッシュを使って `mise` のキャッシュや仮想環境、`uv.lock` 等を保存すればビルド高速化できる

---

## 7. 補足・注意点

* **仕様変更に注意**：mise や uv は活発に開発されているため、将来挙動が変わる可能性があります。
* **mise と uv のバージョン優先順位**：GitHub issue などでは、uv 側が mise 管理バージョンを外部扱いと見なしてしまうケースの報告があります。([GitHub][8])
* **ツール非登録ケース**：もし使いたいツールが mise のレジストリにない場合、GitHub リリースや asdf プラグイン経由で導入できる可能性があります。([Mise-en-place][2])

---

[1]: https://mise.jdx.dev/?utm_source=chatgpt.com "mise-en-place: Home"
[2]: https://mise.jdx.dev/dev-tools/?utm_source=chatgpt.com "Dev Tools | mise-en-place"
[3]: https://mise.jdx.dev/lang/python.html?utm_source=chatgpt.com "Python | mise-en-place"
[4]: https://mise.jdx.dev/installing-mise.html?utm_source=chatgpt.com "Installing Mise | mise-en-place"
[5]: https://mise.jdx.dev/getting-started.html?utm_source=chatgpt.com "Getting Started | mise-en-place"
[6]: https://www.stuartellis.name/articles/mise-en-place/?utm_source=chatgpt.com "mise-en-place for Managing Development Tooling - Stuart Ellis"
[7]: https://mise.jdx.dev/cli/install.html?utm_source=chatgpt.com "mise install - mise-en-place"
[8]: https://github.com/jdx/mise/discussions/4377?utm_source=chatgpt.com "uv + mise and the pipx backend · jdx mise · Discussion #4377 - GitHub"
[9]: https://mise.jdx.dev/cli/ls-remote.html?utm_source=chatgpt.com "mise ls-remote - mise-en-place"

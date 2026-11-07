---
name: matlantis-job-skill
description: Run computational chemistry simulations on Matlantis (PFP/PFCC) by SSHing from the user's local machine and submitting `mtl-bg-job` background jobs. Use this skill whenever the user wants to execute calculations on Matlantis, submit/check `mtl-bg-job` jobs, transfer .cif/.py/.ipynb files to or from the matlantis remote host, or run structure optimizations / adsorption energy / NEB / similar calculations using pfp_api_client / ASECalculator. Trigger on mentions of "Matlantis", "mtl-bg-job", "ssh matlantis", "pfp_api_client", "ASECalculator", 構造最適化・吸着エネルギー計算など、または .cif ファイルを扱う依頼。リモートで動かすことが少しでも示唆されたら積極的に発動すること。
---

# Matlantis SSH ジョブ実行スキル

ローカル（Windows / PowerShell）から Matlantis の Jupyter ワークスペースに SSH 接続し、`mtl-bg-job` を使ってバックグラウンドジョブを投入・監視・回収するためのスキル。

## ユーザー環境の定数

| 項目 | 値 | 確認方法 |
|---|---|---|
| SSH ホスト名 | `matlantis` （`~/.ssh/config` に登録済み） | ローカルの `~/.ssh/config` の `Host` と一致するか。接続・ツール疎通は `ssh matlantis "mtl-bg-job list --phase all"`（「SSH 接続: 必ず PowerShell を使う」の例） |
| リモートホーム | `/home/jovyan` | `ssh matlantis "echo $HOME"`。スクリプト内の入出力パスは後述「スクリプト内のパス取り扱い」のとおり、このホームを前提に絶対パスで書く |
| 利用可能カーネル | `python313` | `ssh matlantis "mtl-bg-job kernels"`（「困ったとき」参照）。一覧と異なると「トラブルシューティング」の kernel エラー行のとおり失敗しうる |
| 既定カーネル | `python313`（特に理由がなければこれ） | 上記 `kernels` の出力に含まれるか。`mtl-bg-job run` では `--kernel python313` を付ける（「ジョブ投入」参照） |
| ローカルモジュール | `/home/jovyan/ase_toolbox` などをホーム直下に配置している（後述の `sys.path` 注意点を参照） | リモートで `ssh matlantis "ls -d ~/ase_toolbox"` などで存在確認。import 失敗時は「トラブルシューティング」の自作モジュール行と「スクリプト内のパス取り扱い」(a) の `sys.path.insert(0, '/home/jovyan')` |

以降のコマンド例は本表の SSH ホスト名・カーネル名に合わせている。値を環境に合わせて書き換えたら、例のコマンドも同様に読み替えること。

## SSH 接続: 必ず PowerShell を使う

**ルール:**
- `ssh` / `scp` を呼び出すときは必ず PowerShell ツールを使う
- ローカル側の `cat`・`grep` 等は通常通り Read / Grep ツールを使えばよい

```powershell
# OK
ssh matlantis "mtl-bg-job list --phase all"

# NG（Bash ツールから実行すると失敗する）
```

## `mtl-bg-job` の全体像

```
mtl-bg-job
├── run      # ノートブック/スクリプトをバックグラウンド実行
├── status   # 個別ジョブの状態確認
├── list     # ジョブ一覧
├── cancel   # ジョブのキャンセル
├── update   # ジョブ情報の更新（メモなど）
└── kernels  # 利用可能カーネル一覧
```

詳細は `ssh matlantis "mtl-bg-job <command> --help"` で確認できる。不確かな場合は推測せず必ず `--help` を見ること。

## ワークフロー

### 1. スクリプト/ファイルをリモートに転送

`scp` を使う。フォルダごと転送する場合は `-r`。

```powershell
# 単一ファイル（ローカルパスは環境に合わせて変更）
scp "my_script.py" matlantis:~/my_script.py

# フォルダごと（CIF などの入力ファイルを同梱したい場合に推奨）
scp -r "my_project" matlantis:~/my_project
```

転送後は `ssh matlantis "ls ~/my_project/"` で内容を確認しておくと安全。

### 2. ジョブ投入: `mtl-bg-job run`

```
mtl-bg-job run [OPTIONS] INFILE OUTFILE
```

- INFILE / OUTFILE は **どちらも Jupyter ワークスペース内（`/home/jovyan` 配下）** に置く必要がある
- `.py` スクリプトの場合 `--kernel` は **必須**
- OUTFILE には stdout / stderr が書き出される（ジョブの「ログ」になる）

```powershell
# 推奨: 入出力ともにプロジェクトフォルダ内にまとめる（カーネルは上表の既定に合わせる。迷ったら mtl-bg-job kernels で確認）
ssh matlantis "mtl-bg-job run ~/my_project/run.py ~/my_project/stdout.log --kernel python313"
```

主なオプション:
- `--kernel <name>`: カーネル指定（`.py` で必須）
- `--note "..."`: ジョブにメモを付与（最大 100 文字、後で `list` で見やすくなる）
- `-p KEY VALUE`: ノートブックへのパラメータ注入（スクリプトでは不可）
- `--format json`: スクリプトから扱うときに便利

戻り値は `job_id`。これを必ず控える。

### 3. 【重要】スクリプト内のパス取り扱い

`mtl-bg-job run` はスクリプトを `/home/jovyan/.local/share/matlantis/jobs/mtl_bg_run_xxxxxxxx.py` のようなテンポラリ位置にコピーして実行する。そのため:

#### (a) ホーム直下の自作パッケージは、そのままでは import できない

ジョブ実行時はスクリプトが一時ディレクトリから走るため、ホームに置いた自作モジュールがパスに含まれない。スクリプト先頭で `sys.path` にリモートホームを追加する（ホームが `/home/jovyan` でない環境では実際のパスに合わせる）:

```python
import sys
sys.path.insert(0, '/home/jovyan')

# 以降、~/ase_toolbox/... などが import 可能になる
from ase_toolbox.HandleAtoms import fix_layers
```

これを忘れると `ModuleNotFoundError` で `failed` になる。

#### (b) 入出力ファイルは絶対パスで参照する

カレントディレクトリは実行時テンポラリなので、相対パスや `__file__` ベースのパス解決は当てにならない。プロジェクトフォルダの絶対パスを直書きする:

```python
from pathlib import Path
FOLDER = Path('/home/jovyan/my_project')
CIF_FILE = FOLDER / 'input.cif'
LOG_FILE = FOLDER / 'result.log'
OUT_CIF  = FOLDER / 'optimized.cif'
```

#### (c) ロギングの推奨パターン

OUTFILE にも残るが、構造化ログを別途ファイル出力しておくと後で扱いやすい:

```python
import logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s - %(message)s',
    handlers=[
        logging.FileHandler(LOG_FILE),
        logging.StreamHandler(sys.stdout),
    ],
)
log = logging.getLogger(__name__)
```

### 4. ステータス確認

```powershell
# 個別ジョブ
ssh matlantis "mtl-bg-job status <job_id>"

# 直近のジョブ一覧（status を問わない場合）
ssh matlantis "mtl-bg-job list --phase all"
```

`status` が `running` ならまだ実行中。`done` / `failed` / `canceled` で終了。

ジョブが `failed` した場合は、まず OUTFILE（stdout.log 等）の中身を `cat` で確認すること。原因のほぼ全てがそこに出る。

```powershell
ssh matlantis "cat ~/my_project/stdout.log"
```

### 5. 結果の回収

```powershell
# プロジェクトフォルダごと持ち帰るのが楽
scp -r matlantis:~/my_project "my_project-result"
```

回収後、ローカルの Read / Glob ツールで内容を確認。
副産物として `.ipynb_checkpoints/` が混ざることがあるが無視してよい。

## 完全な手順テンプレート

新規シミュレーションを依頼されたときの標準手順:

1. ローカルにプロジェクトフォルダを作る（例: `Desktop\<task-name>\` に CIF や入力ファイルを置く）
2. その中に Python スクリプトを作る。冒頭で `sys.path.insert(0, '/home/jovyan')`、入出力は `/home/jovyan/<task-name>/` の絶対パスで書く
3. `scp -r` でフォルダをアップロード
4. `ssh matlantis "ls ~/<task-name>/"` でアップロード確認
5. `mtl-bg-job run ~/<task-name>/run.py ~/<task-name>/stdout.log --kernel python313` でジョブ投入（カーネルは上表と異なる場合は `mtl-bg-job kernels` で確認）、`job_id` を控える
6. `mtl-bg-job status <job_id>` でステータス確認（必要なら数十秒待つ）
7. `done` なら OUTFILE と独自ログを `cat` で確認
8. `scp -r matlantis:~/<task-name> "<task-name>-result"` などで回収（保存先は任意）

## 計算系の典型パターン

### 構造最適化（バルク / 表面）

```python
import sys; sys.path.insert(0, '/home/jovyan')
from pathlib import Path
import pfp_api_client
from pfp_api_client.pfp.calculators.ase_calculator import ASECalculator
from pfp_api_client.pfp.estimator import Estimator, EstimatorCalcMode
from ase.io import read, write
from ase.optimize import FIRE
from matlantis_features.ase_ext.optimize import FIRELBFGS # Matlantis環境ではより効率的なFIRELBFGSが使用できる

FOLDER = Path('/home/jovyan/<task-name>')
atoms = read(FOLDER / 'input.cif')
# model_version は必ず明示する（省くと既定の版になり、新しい版は明示しないと使われない。版で結果が変わりうる）
estimator = Estimator(calc_mode=EstimatorCalcMode.R2SCAN_PLUS_D3, model_version="v8.0.0")
atoms.calc = ASECalculator(estimator)  # Estimator は Calculator ごとに新しく作る（共有しない）
print(f"pfp_api_client: {pfp_api_client.__version__}, model: {estimator.model_version}, mode: {estimator.calc_mode}")

opt = FIRELBFGS(atoms, logfile=str(FOLDER / 'opt.log'))
opt.run(fmax=0.01, steps=500)

write(FOLDER / 'optimized.cif', atoms)
print(f"E = {atoms.get_potential_energy():.6f} eV")
```

## トラブルシューティング

| 症状 | 原因 | 対処 |
|---|---|---|
| `ModuleNotFoundError: No module named 'ase_toolbox'` など | 実行ディレクトリがホームではない | スクリプト冒頭に `sys.path.insert(0, '/home/jovyan')`（実際のホームに合わせる） |
| `FileNotFoundError` で .cif が見つからない | 相対パスで書いている | 絶対パス（`/home/jovyan/...`）に書き換え |
| Bash ツール経由で `ssh: ... .bat: not found` | Bash 経由だと SSH ホスト解決が失敗する | PowerShell ツールから実行する |
| `mtl-bg-job run` で kernel エラー | `.py` で `--kernel` を指定していない、または名前が環境と不一致 | `mtl-bg-job kernels` で正しい名前を確認し `--kernel <name>` を付ける |
| 結果ログが空 | ジョブがまだ起動直後 | 数秒〜数十秒待ってから再確認 |
| `MultiCalculatorUseDetected` で落ちる | 1 つの `Estimator` を複数の `ASECalculator`（吸着系・清浄面・気相分子など）で共有している | Calculator ごとに `Estimator(...)` を新しく作る |
| 以前と同じ計算なのに値が 0.1 eV 程度違う | `model_version` を指定せず、既定の版が使われた（または版を変えた） | `Estimator(..., model_version="vX.Y.Z")` で明示し、版をログに出す |

## 危険操作・確認すべきこと

- リモートのファイルを削除（`ssh matlantis "rm ..."`）する場合は事前にユーザー確認
- `mtl-bg-job cancel` を実行する前にも確認
- 既存ジョブの結果を `scp` でローカルの既存フォルダに上書きする場合は別名（`-result` サフィックス等）にして衝突を避ける

## 困ったとき

- 各サブコマンドの正確な仕様: `ssh matlantis "mtl-bg-job <cmd> --help"`
- カーネル一覧: `ssh matlantis "mtl-bg-job kernels"`
- リモートのファイル一覧: `ssh matlantis "ls ~/<folder>/"`

推測で動かさず、不明点は必ずリモートで確認すること。

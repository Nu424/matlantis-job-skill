# matlantis-job-skill

SSH経由でMatlantisに接続し、`mtl-bg-job` コマンドを使用してバックグラウンドジョブを管理するためのAgent Skillsです。

## 使い方
1. (前提) MatlantisへSSH接続できる環境を整備する
    - 公式のFAQを参照してください: [Matlantis FAQ](https://matlantis.zendesk.com/hc/ja/articles/40524152374169-SSH接続-設定方法-2026-3-6-更新)
    - ない場合: [Matlantis FAQのトップ](https://matlantis.zendesk.com/hc/ja)
2. `SKILL.md` を配置する
    - 使用するAIエージェントに合わせて設定してください。
    - `npx skills add` を使用する場合、以下のコマンドで導入できます
    ```bash
    npx skills add Nu424/matlantis-job-skill
    ```
3. 適宜SKILL.mdを微調整する
    - `## ユーザー環境の定数` の値は、自分の SSH 設定・Matlantis 環境に合わせて変更してください（確認方法列で検証可能）。
4. 使ってみる
    - 例: 既存のシミュレーションコードを実行してもらう
    ```
    @run.py このコードをMatlantis上で実行して、結果を教えてください。
    ```
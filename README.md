# Pivotal Container Serivce - First Contact 2019/08/30 - ハンズオン資料
## 注意事項
- 本資料は8/30当日のハンズオン向けの手順ならびに設定情報等を含みます。手順については再利用可能ですがハンズオン環境はセッション終了後に削除する為使えなくなります。
- 今レポジトリは期間限定でpublicにしておりますが、ワークショップ数日後にprivateに切り替えます。それまでにcloneして頂く事は構いませんが、pksのcliバイナリにつきましてはご使用をお控えください。
- 参加者向けの情報ですので情報の共有はお控えください。
## 事前準備
- PKSのAPIに対してssh経由でアクセスする必要があります。Windows環境の場合にはGit for Windows ( https://gitforwindows.org/ )等を事前インストールしてください。
## クライアントツールのインストール（kubectl）
- https://github.com/shinichi-hashitani/pks-workshop-2019-aug/tree/master/kubectl にあるファイルのうち自分の端末OSと互換性のあるファイルを取得し"kubectl"という名前で保存。（Windowsの場合kubectl.exe）
- 保存したファイルを適当な場所に保存しパスを設定。
- 新しいターミナルで"kubectl -h"と実行。ヘルプ情報が表示される事を確認。
## クライアントツールのインストール（pks cli）
- https://github.com/shinichi-hashitani/pks-workshop-2019-aug/tree/master/pks-cli にあるファイルのうち自分の端末OSと互換性のあるファイルを取得し"pks"という名前で保存。（Windowsの場合pks.exe）
- 保存したファイルを適当な場所に保存しパスを設定。
- 新しいターミナルで"pks -h"と実行。ヘルプ情報が表示される事を確認。

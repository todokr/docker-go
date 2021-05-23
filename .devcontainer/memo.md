# 骨子

## Linux コンテナとは何者か

- コンテナなんて「モノ」はない
- コンテナ = 隔離された特別なプロセス

## 何を隔離している？
- プロセス、ファイル構造、ユーザーID、グループIDなどの「データ」 -> Namespaceという仕組みで隔離
- CPU、メモリなどの「コンピューティング資源」 -> Control Group（cgroup）という仕組みで隔離

## Namespace
- まさしく名前空間をつくるもの
- `unshare` コマンドで作られる
- 分離させたい対象によっていくつかの機能がある
  - PID名前空間: プロセスIDの分離。異なる名前空間同士では、同一のプロセスIDを持つことが可能になる。
  - ユーザ名前空間: ユーザーID, グループIDの分離。
  - UTS名前空間: ホスト名, ドメイン名の分離。
  - ネットワーク名前空間: ネットワークデバイス, ポート, ルーティングテーブル, ソケットなどの分離。
  - マウント名前空間: マウントの集合, 操作。ファイルシステムのマウントポイントを分離する。Namespace 内の mount / umount が他の Namespace に影響を与えないようにする。
  - IPC名前空間: SysV IPCオブジェクト, POSIXメッセージキューなどプロセス間通信のためのリソースの分離。

### プロセスIDの分離
`unshare` の `--pid` オプションでプロセスIDの空間を分離してみる。

1. 親と子用のタブを開く
1. 子になる方のタブで `sudo unshare --pid --fork --mount-proc /bin/bash` 
  1. 参考: https://tech.retrieva.jp/entry/2019/04/16/155828
1. 親のタブで `sleep 60 &` 
1. 親のタブで `ps aux | grep sleep` 。 いまのsleepのプロセスが確認できる
2. 子のタブで `ps aux | grep sleep` 。 親のプロセスは見えない。隔離されていることがわかる

### ちなみに、subshellは親PIDが見える
1. `(sleep 5 && ps aux | grep sleep)&` # サブシェルで5秒後にgrep
1. `sleep 40` # サブじゃないshellでsleep
1. サブじゃないshellでやったsleepのPIDが見える

### UTS名前空間の分離
`unshare` の `-u` オプションでUTS名前空間を分離してみる。

まずは普通にhostnameを変更してみる例。

1. タブAで `hostname host` を実行
1. タブAで `hostname` を実行。さっき設定したホスト名である `host` が表示される
1. タブBで `hostname` を実行。こちらでも `host` が表示される。つまり、 UTS名前空間が分離されていない（当たり前だけど）

次に、 `unshare -u` でUTS名前空間を分離する。

1. タブAで `sudo unshare -u /bin/bash` を実行
1. タブAで `hostname another_host` を実行
1. タブAで `hostname` を実行。 `another_host` と表示される
1. タブBで `hostname` を実行。 タブAでホスト名を設定したのに、 `host` と表示される。つまり、UTS名前空間が分離できている。

## unshare して立ち上がったプロセス
- これこそがコンテナの正体
- VirtualBoxなどのホストOS型仮想化とは異なり、仮想マシンやゲストOSが不要なので軽量、立ち上がりが高速
- コンテナとホストはLinuxカーネルを共有
  - ちなみにMacはLinuxカーネルを持っていないが...？
  - 実はDocker for Macの正体（の一部）は仮想マシンで動くLinuxなんだ！
  - Docker for MacでDockerを動かす = DockerのコンテナはホストOSであるOSXではなく、Docker for Macの仮想マシンで動いているLinuxとカーネルを共有する


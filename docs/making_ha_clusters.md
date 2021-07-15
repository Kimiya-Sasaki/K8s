## kubeadm による HA Clusters の構築
### 1. 前提条件
  - Linux（ここでは Ubuntu 20.04 LTS を使用）
    - Raspberry PI 4 でも可能 
  - CRI 準拠のコンテナエンジンがインストールされている（Dockerなど） 
    - [Docker のインストール](https://kubernetes.io/ja/docs/setup/production-environment/container-runtimes/)
    - 少し下にスクロールするとある
  - kubeadm, kubelet, kubectl がインストールされている 
    - [kubeadm のインストール](https://kubernetes.io/ja/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
  - swap が off になっている 
    - `# swapoff -a & free` → 以下の様に Swap が 0 になっていることを確認する
<pre>
              total        used        free      shared  buff/cache   available
Mem:        6085584      511076     3799944        1280     1774564     5359340
Swap:             0           0           0
</pre>
  - 各マシンが互いに名前解決できる（/etc/hosts）
    - IP で直接設定していする場合は必要ない 
  - 各マシンが通信可能
  - 各マシンの時間が同期されている
<pre>
$ sudo vi /etc/systemd/timesyncd.conf  
  NTP=ntp.nict.jp # この行を追加
$ sudo systemctl restart systemd-timesyncd.service 
</pre>
  - cgourp-driver: kubelet, docker ともに systemd を前提とする
    - kubelet は kubeadam を使うと Default で systemd になる 
    - Docker の Default は cgroupfs だが Multi Cluster 構築のときに kubeadm でエラーになるので systemd に変更
      - `kubectl apply -f https://docs.projectcalico.org/v3.14/manifests/calico.yaml`
  - 基本的に作業は全て root で行っている（一部 user とは作業が異なる）
    -  `$ sudo passwd root`

### 2. High Availability/HA Clusters 環境の構築手順
1. Create LB : ロード バランサ 
2. Create Basic Control-Plane Node : 基本の Control-Plane (Master) 
3. Install CNI : CNI のインストール 
4. Create Another Control-Plane Nodes : その他の Control-Planes 
5. Create Worker Nodes : ワーカー ノード 

### 3. HA Clusters 構成（２部構成） 
今回は前者（Stacked etcd）の構成を取る
![Stacked Etcd](../imgs/stackedetcd.png)
![External Etcd](../imgs/extenaletcd.png)

#### 1. Create LB
#### 2. Create Basic Control-Plane Node : 基本の Control-Plane (Master) 
#### 3. Install CNI : CNI のインストール 
- `kubectl get nodes` → Ready になっていることを確認 
- エラーが発生したときは以下のコマンドで環境をチェック
<pre>
# nc -v &lt;IP of LB&gt; 6443 
  Connection to &lt;IP of LB&gt; 6443 port [tcp/*] succeeded! → このメッセージが出力されると成功 
# systemctl restart kubelet → 全ての Control-Plane 上で 
# systemctl restart haproxy → haproxy をインストールしたマシン上で 
</pre>
#### 4. Create Another Control-Plane Nodes : その他の Control-Planes 
#### 5. Create Worker Nodes : ワーカー ノード 


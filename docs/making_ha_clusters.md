## kubeadm による HA Clusters の構築
### 1. 前提条件
  - Linux（ここでは Ubuntu 20.04 LTS を使用）
    - Raspberry PI 4 でも可能 
  - 基本的に作業は全て root で行っている（一部 user とは作業が異なる）
```
sudo passwd root
```
  - HA Clusters の動きを厳密に検証したければ、最低でも下記の合計7台のマシンを用意する必要がある
    - Load Balancer : 1台
    - Control-Plane : 3台
    - Woker Nodes   : 3台
  - CRI 準拠のコンテナ エンジンがインストールされている（Dockerなど） 
    - [Docker のインストール](https://kubernetes.io/ja/docs/setup/production-environment/container-runtimes/) - 少し下にスクロール
    - Raspberry PI にインストールするときは、リポジトリの追加の場所で arch=amd64,arm64 とする（amd64は削除して良い）
  - kubeadm, kubelet, kubectl がインストールされている

iptables がブリッジを通過するトラフィックを処理できるようにする
```
cat <<EOF > /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system
```
iptables が nftables バックエンドを使用しないようにする
```
apt-get install -y iptables arptables ebtables

update-alternatives --set iptables /usr/sbin/iptables-legacy
update-alternatives --set ip6tables /usr/sbin/ip6tables-legacy
update-alternatives --set arptables /usr/sbin/arptables-legacy
update-alternatives --set ebtables /usr/sbin/ebtables-legacy
```
kubeadm、kubelet、kubectl のインストール
```
apt-get update && sudo apt-get install -y apt-transport-https curl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
apt-get update
apt-get install -y kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubectl
```
- swap を以下の様に 0 にする
```
swapoff -a & free
```
<pre>
              total        used        free      shared  buff/cache   available
Mem:        6085584      511076     3799944        1280     1774564     5359340
Swap:             0           0           0
</pre>
  - 各マシンが互いに名前解決できる（/etc/hosts）
    - IP で直接設定する場合は必要ない 
  - 各マシンが通信可能
  - 各マシンの時間が同期されている
```
vi /etc/systemd/timesyncd.conf
# NTP=ntp.nict.jp を追加
systemctl restart systemd-timesyncd.service 
```
  - cgourp-driver: kubelet, docker ともに **systemd** を前提とする
    - kubelet は kubeadam を使うと Default で systemd になる 
    - Docker の Default は cgroupfs だが HA Cluster 構築のときに kubeadm でエラーになるので [Docker のインストール](https://kubernetes.io/ja/docs/setup/production-environment/container-runtimes/) で systemd に変更している
### 2. HA Clusters 構成（２部構成） 
今回は前者（stacked etcd）の構成を取る。external etcd にする場合はさらに 3台のマシンが必要になる。

<img src="../imgs/stackedetcd.png" width="649px">
<img src="../imgs/extenaletcd.png" width="640px">

### 3. High Availability/HA Clusters 環境の構築手順
1. Create LB : ロード バランサ → LB として稼働するマシンだけ
2. Create Basic Control-Plane Node : 基本の Control-Plane (Master) 
3. Install CNI : CNI のインストール 
4. Create Another Control-Plane Nodes : その他の Control-Planes 
5. Create Worker Nodes : ワーカー ノード 
#### 1. Create LB
HAProxy のインストール : この作業は Load Balancer として稼働させるマシンだけに行う。
また LB として稼働するマシンは、それ以外の作業は不必要。
```
apt update 
apt install haproxy 
```
設定ファイルを編集する
```
vi /etc/haproxy/haproxy.cfg
```
以下の内容を最下行に追加する
```
frontend kubernetes 
        mode tcp 
        option  tcplog 
        bind <自分の IP Address>:6443 # IP は * でも良い 
        default_backend kubernetes-master-nodes 

backend kubernetes-master-nodes 
        mode tcp 
        balance roundrobin 
        server  <machine name> <IP of Master Node>:6443 check fall 3 rise 2 # <machine name> e.g, master01
        server  <machine name> <IP of Master Node>:6443 check fall 3 rise 2
        server  <machine name> <IP of Master Node>:6443 check fall 3 rise 2
```
リスタートする
```
systemctl daemon-reload
systemctl restart haproxy
```
エラーが発生したとき以下のコマンドで調査 
```
journalctl -u haproxy.service --since today --no-pager 
```
#### 2. Create Basic Control-Plane Node : 基本の Control-Plane (Master) 
- Raspberry PI で動かすときは下記の操作が必要
```
vi /boot/firmware/cmdline.txt
```
以下の内容を最後に付け加える。保存した後はリブート。
```
cgroup_memory=1 cgroup_enable=memory
```
- 以下のコマンドで cgroup の memory が 1 になっていることを確認
<pre>
cat /proc/cgroups

#subsys_name    hierarchy       num_cgroups     enabled
cpuset  9       10      1
cpu     2       103     1
cpuacct 2       103     1
blkio   4       103     1
memory  8       159     1  # ここを確認！
devices 11      103     1
<以下省略>
</pre>
- `kubeadm init --config *.yaml` → この書式では HA Clusters 用の出力がなされないので以下の書式を使う
```
kubeadm init --control-plane-endpoint <IP of LB>:6443 --upload-certs
```
- 2時間で証明書が無効になるので、そのときは以下のコマンドを使う
```
kubeadm init phase upload-certs --upload-certs 
```
- 二通りの kubeadm join が出力されるのでコピーしておく 
  - 上は Control-Plane (Master) を登録するときに使用 
  - 下は Worker Node を登録するときに使用
- KUBECONFIG 環境変数を設定する。因みにリモート アクセスする場合はこの環境構築は必要になるので ~/.bashrc に書き込んでおくと便利
```
export KUBECONFIG=/etc/kubernetes/admin.conf
```
#### 3. Install CNI : CNI のインストール 
```
kubectl apply -f https://docs.projectcalico.org/v3.14/manifests/calico.yaml
```
- `kubectl get nodes`
STATUS: Ready になっていることを確認
<pre>
NAME            STATUS   ROLES                  AGE   VERSION
master01        Ready    control-plane,master   10h   v1.21.2
</pre>
#### 4. Create Another Control-Plane Nodes : その他の Control-Planes
- kubeadm init の出力でコピーした**上の** kubeadm join を該当するマシン上で実行する 
- 事前に前提条件をすべて満たしておくこと
- Option に --control-plane --certificate-key が**ある**ことを確認 
<pre>
kubeadm join &lt;IP of LB&gt;:6443 --token XXXXXX --discovery-token-ca-cert-hash XXXXXX --control-plane --certificate-key XXXXXX
</pre>
- Control-Plane 上で join できたかを確認する
```
kubectl get nodes
```
STATUS: Ready になっていることを確認
<pre>
NAME            STATUS   ROLES                  AGE   VERSION
master01        Ready    control-plane,master   10h   v1.21.2
master02        Ready    control-plane,master   10h   v1.21.2
master03        Ready    control-plane,master   10h   v1.21.2
</pre>
#### 5. Create Worker Nodes : ワーカー ノード 
- kubeadm init の出力でコピーした**下の** kubeadm join を該当するマシン上で実行する
- 事前に前提条件をすべて満たしておくこと（kubectl は使えない）
- Option に ---control-plane --certificate-key が**ない**ことを確認
<pre>
kubeadm join &lt;IP of LB&gt;:6443 --token XXXXX --discovery-token-ca-cert-hash XXXXXX
</pre>
#### その他
- 動作確認 on Control-Plane
```
kubectl get nodes      # STATUS: Ready 
kubectl get pods –A    # STATUS: Running 
```
- 接続が上手くいかなかったとき

LB 以外の全てのマシン上で kubelet を restart
```
systemctl restart kubelet
```
LB をインストールしたマシン上で
```
systemctl restart haproxy
```
Control-Plane のマシン上で
```
export KUBECONFIG=/etc/kubernetes/admin.conf
```
- 環境のリセット
```
rm -rf /etc/kubernetes/* 
rm -rf /var/lib/etcd/* 
kubeadm reset
```

https://kubernetes.io/docs/concepts/cluster-administration/networking/#how-to-implement-the-kubernetes-networking-model


### kube-proxy

Serviceは `kube-proxy` とそれを支える機能によって実現されている。

各ノードで動作する `kube-proxy` の主な役割は、ネットワークの情報を `kube-apiserver` から取得して自ノードが持つ `iptables` を動的に更新していくことだ。iptablesを書き換えることによって、Linuxカーネルがリクエストをポッドに伝送できるようになる。（エンドポイントレコードが作成されたとき、各ノードのiptablesも書き換わる）




`iptables` はサービスの仮想IPアドレスを紐付くポッドIPアドレスに書き換える役割を持つが、そのポッドが動くノードを特定することはできない。ポッドIPアドレスに書き換えられたパケットは `flanneld` によってそのポッドが動くノード宛のパケットになり、伝送される。


#### クラスタ内部での通信

あるPodがService（別のPod）と通信


にと通信する時、パケットはまずiptablesによって解釈される。


する時、リクエストはまずiptablesに届き、正確なIPアドレスを伝ってリクエストが伝送される。（これは別ノードのポッドかもしれない）
# KubernetesとJenkinsCI

@ladicle

---

# 目次

1. 環境構築 - Jenkinsとk8sの構成-
2. 試験 - CIの流れと設定ファイル -
3. 所感 - Pros/Cons -

---

# 1. 環境構築

---

# 現在の環境

![Architecture](img/architecture.png)

---

# 構築方法

各VMのOS: Ubuntu14.04

## Docker
* [Jenkins](https://hub.docker.com/_/jenkins/)
* [ImageRegistry](https://docs.docker.com/registry/)

## Ansible
* [Kubernetes](https://github.com/kubernetes/contrib/pull/802)

---

# Jenkinsの構築

## デフォルト機能を使った拡張
* `plugin.txt`へ使用するPluginの追加
* 起動オプションでタイムゾーンを指定

        JAVA_OPTS='-Duser.timezone=Asia/Tokyo'

---

#### このままでは作って捨てたら戻ってこないので...
## Dockerfileに追記が必要なもの

---

## Kubernetes設定

* kubectlのバイナリインストール 
* `$HOME/.kube`に設定ファイル追加

    Ansibleで構築した場合は各Nodeの以下パスを.kubeへ
    /etc/kubernetes/kubectl.kubeconfig
    
---

## Docker設定

* Dockerコマンドのインストール
* RemoteAPIようの鍵を追加
 * ca.pem,
 * server.pem, 
 * server-key.pem
 
    DOCKER_HOST, DOCKER_TLS_VERIFY, DOCKER_CERT_PATHは
    Jenkinsの環境変数に指定しておくと便利
 
---

## ユーザ情報/全体設定/パスワード

JenkinsGUIで設定後
生成ファイルをDockerfileへ組み込む

  * $JENKINS_HOME/users/*
  * $JENKINS_HOME/config.xml
  * $JENKINS_HOME/credentials.xml

---

## JOB設定

[Jenkins Job Builder](http://docs.openstack.org/infra/jenkins-job-builder/)
使ってJob設定をYAML管理

---

## k8sの構築

#### Playbookに+αした部
`group_vars/all.yaml`に以下を追加
* ImageRegistryのアドレス
* SSHユーザ名

#### 構築後の手動設定
[SecurityContext](http://kubernetes.io/docs/user-guide/security-context/)でImageRegistryのログイン情報を追加

    kubectl create secret docker-registry <key-name> --docker-server=<registry address> --docker-username=<login user> --docker-password='<login password>' --docker-email='<email>'

---

# 2. 試験

---

## 試験の流れ

![testflow](img/testflow.png)

---

## 文法チェック

チェック対象のファイルを各コマンドが実行可能なコンテナにマウントさせて試験している

* k8sのManifest: Pythonの[yamllint](https://pypi.python.org/pypi/yamllint/0.5.1)
* Dockerfile: JSのdockerlint

> Manifestの文法チェックでは、実際にcreateさせるかで迷ったが
> 細かい部分は実際に中入って対象のテストを叩かないとわからないので
> 最小限のYAMLのフォーマットチェックに留めた

---

## Unit Test

JenkinsのWorkspaceをマウントしたコンテナ起動 -> 単体テスト実行 -> お片付け

---

## Integration Test

Dockerイメージのビルド -> 前回のお片付け -> コンテナで全コンポーネント起動 -> シナリオテスト実行

---

## Manifestの拡張子

JSONとYAMLのどちらも選択できるが、
混在するのは論外なのでチームにアンケートとった。
=> 満場一致で**YAML**に決定。

> JSON, 読むのは`jq`あるので良いが書くのがつらい
> YAMLの`jq`的な存在[yq](https://github.com/abesto/yq)も存在する。ちょっと便利

---

## Manifestへ変数の埋め込み

1. パスワード含め、変数はJenkinsから操作しやすいように環境変数に代入
2. 環境変数を以下のコマンドで展開してall-in-oneのファイルを作成

        printf "cat <<++EOS\n%s\n++EOS\n" "$(cat *.yaml)" | sh > all-in-one.yaml
        
> 管理しやすいようにYAMLファイルは各コンポーネントごとにserver/deploymentを作成している
> ただし、create/deleteしやすいよう変数展開時にall-in-one.yamlへまとめてる

---

## ConfigMapとSecurityContexts

最低限必要なPrivateRegistryからのイメージPull用のSecurityContexts以外
ConfigMapやSecurityContextsは使用していない
これはJenkinsで動作させる以上、環境変数化したほうがシンプルかつ柔軟に操作できるため

      configMap:
        name: redis-volume-config
        items:
          - path: "etc/redis.conf"
            key: redis.conf

> configMapが上記のように一つづつ指定するのではなくpathで一括指定できたら
> コンポーネント間で共通の環境変数とかに利用しやすいんですが...
> テンプレートエンジンかますとか以外で良い方法募集中!

---

## [おまけ] Jenkins -KubernetesPlugin-

k8sを操作するPluginは観測する限り[KubernetesPlugin](https://wiki.jenkins-ci.org/display/JENKINS/Kubernetes+Plugin)のみ
これはJenkinsのSlaveをKubenetes上にのせるものなので自前Manifestの操作には向かない

> JenkinsJobBuilderも未対応

---

# 3. 所感

---

## Pros
* 1ファイルにまとめられるので簡単にcreate/deleteできる
 * kubectlコマンドの引数がシンプルで、試験用のスクリプトを後から見やすい
 * パラメータの受け渡しも必要なく、ファイル指定のdeleteコマンドだけで試験の後片付けできるのがよい
* Slackコミュニティーが活発&寛容で、`#kubernetes-user`に質問投げると誰かしら答えてくれる

---

## Cons

覚えることおおい

---

## その他

Kubernetesのレポジトリ、いろんなBotが住んでいて便利
(特にRVを煽ってくるのがよい)

---

# Thank you!

[![application ci](https://github.com/mosuke5/sample-validation-admission-webhook/actions/workflows/test.yaml/badge.svg)](https://github.com/mosuke5/sample-validation-admission-webhook/actions/workflows/test.yaml)

# Sample Validation Admission Webhook
This repo was modified based on [original repo](https://github.com/mosuke5/sample-validating-admission-webhook)

参考URL
- https://blog.mosuke.tech/entry/2022/05/15/admission-webhook-1/
- https://blog.mosuke.tech/entry/2022/05/15/admission-webhook-2/

## 何をするものか？
AKS(Azure Kubernetes Servcie)やARO(Azure RedHat Openshift)の環境で、ロードバランサタイプのServiceオブジェクトを作る際に、以下のようにannotationsを記述することで内部LB(ILB)にルールを構成することができる仕様があります。

```
apiVersion: v1
kind: Service
metadata:
  name: hellosvc
  namespace: sampleproj
  annotations:
    service.beta.kubernetes.io/azure-load-balancer-internal: "true"
spec:
...
```

このとき、annotationsの記述漏れがあるとパブリックLB(ELB)にinboundのアクセスエンドポイントができてしまうため、kubectl apply時にyamlをvalidate（検証）して、Service作成時に上記のILBのannotationがない場合、denyします

## 動作確認手順
★印はoriginal repoのソースを改変したものです

### webhookサーバ(go)の実装★
https://github.com/mosuke5/sample-validating-admission-webhook
をcloneしてserver.goを改変しています。許可・拒否のロジックを変更したい場合はこのコードを変更してください。

### 単体テストの実行
testdata/req1.jsonはテスト用データ(ILB annotations付)、req2はannotationsなしテストデータです。
```
$ go run server.go -server-cert=./tmp/server.crt -server-key=./tmp/server.key -body-dump
$ curl -s -k -H 'Content-Type: application/json' -XPOST https://localhost:8443/runasuser-validation -d @testdata/req1.json | jq .
```

### webhookサーバのコンテナビルド/push
Dockerfileを用いてbuildして自分のrepoにpushしておく(あとでdeploy.yamlでこのイメージをデプロイします）
``` 
$ docker login
$ docker build -t tetsuyasodo/sample-validating-admission-webhook:main .
$ docker push tetsuyasodo/sample-validating-admission-webhook:main
```

### TLS証明書の作成
```
$ mkdir tmp; cd tmp
$ openssl genrsa 2048 > server.key
$ openssl req -new -key server.key -out server.csr  //JP,Tokyo,"mywebhook.mynamespace.svc.cluster.local"を指定
$ echo "subjectAltName = DNS:mywebhook.mynamespace.svc, DNS:mywebhook.mynamespace.svc.cluster.local" > san.txt
$ openssl x509 -req -days 365 -in server.csr -signkey server.key -out server.crt -extfile san.txt
$ openssl x509 -text -noout -in server.crt  //X509v3 SANが設定されているか確認する
```

### 証明書と鍵をsecret/tlsとしてデプロイ
```
$ kubectl create ns mynamespace
$ kubectl create secret tls mywebhook-secret --key server.key --cert server.crt -n mynamespace
```

### webhookサーバイメージを持つPodをデプロイ★
docker imageをpullしています
```
$ kubectl apply -f manifests/deploy.yaml -n mynamespace
$ kubectl get pod,service -n mynamespace
```

### validatingwebhookconfigurationをデプロイする★
cluster-wide resourceとして展開するとwebhookが有効化します
```
$ sed  "s/BASE64_ENCODED_PEM_FILE/$(base64 -w 0 ./tmp/server.crt)/g" manifests/validatingwebhookconfiguration.yaml.template | kubectl apply -f -
```
※注意：元blogではbase64だけ使っているが改行があってsedが失敗するので"-w 0"で改行をしなくする必要がある

### リソース展開時の動作確認
svc1.yamlはannotationが含まれておりデプロイに成功しますが、svc2.yamlはannotationsがないため拒否されることが確認できます
```
$ kubectl create ns user-foo
$ kubectl apply -f svc1.yaml
service/hellosvc created
$ kubectl apply -f svc2.yaml
Error from server: error when creating "svc2.yaml": admission webhook "mywebhook.mynamespace.svc.cluster.local" denied the request: Annotation is required in user namespace.
```

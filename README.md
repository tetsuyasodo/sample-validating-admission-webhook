[![application ci](https://github.com/mosuke5/sample-validation-admission-webhook/actions/workflows/test.yaml/badge.svg)](https://github.com/mosuke5/sample-validation-admission-webhook/actions/workflows/test.yaml)

# Sample Validation Admission Webhook
This repo is modified from [original repo](https://github.com/mosuke5/sample-validating-admission-webhook)

https://blog.mosuke.tech/entry/2022/05/15/admission-webhook-1/

https://blog.mosuke.tech/entry/2022/05/15/admission-webhook-2/

をもとに、user namespaceのserviceでILBのannotationがない場合、denyする仕組みを作る（ARO RA p.51の実装）

★は元repoのソースを改変したもの

### webhookサーバ(go)の実装
https://github.com/mosuke5/sample-validating-admission-webhook
をcloneしてserver.goを改変★

### 単体テストは以下のようにする
testdata/req1.jsonはテスト用データ(ILB anno付)、req2はannoなし
```
$ go run server.go -server-cert=./tmp/server.crt -server-key=./tmp/server.key -body-dump
$ curl -s -k -H 'Content-Type: application/json' -XPOST https://localhost:8443/runasuser-validation -d @testdata/req1.json | jq .
```

### Dockerfileが含まれるのでbuildして自分のrepoにpushしておく(あとでdeploy.yamlで使う）
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

### webhookサーバイメージを持つPodをデプロイ(docker imageをpullしてくる)★
```
$ kubectl apply -f manifests/deploy.yaml -n mynamespace
$ kubectl get pod,service -n mynamespace
```

### validatingwebhookconfigurationをデプロイする(cluster-wide resource)
```
$ sed  "s/BASE64_ENCODED_PEM_FILE/$(base64 -w 0 ./tmp/server.crt)/g" manifests/validatingwebhookconfiguration.yaml.template | kubectl apply -f -
```
※注意：元blogではbase64だけ使っているが改行があってsedが失敗するので"-w 0"で改行をしなくする必要がある

### 動作確認
```
$ kubectl create ns user-foo
$ kubectl apply -f svc1.yaml
service/hellosvc created
$ kubectl apply -f svc2.yaml
Error from server: error when creating "svc2.yaml": admission webhook "mywebhook.mynamespace.svc.cluster.local" denied the request: Annotation is required in user namespace.
```

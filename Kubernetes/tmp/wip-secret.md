# シークレット（Secret）

`service-account-token`

ServiceAccountが作成されたら、Token ControllerによってAPIアクセスするための `Secret` が作成される。（Tokenなどが保存されている）

```
$ kubectl create serviceaccount test
$ kubectl get secret
```

PodはServiceAccountと紐付いて起動される

https://qiita.com/knqyf263/items/ecc799650fe247dce9c5

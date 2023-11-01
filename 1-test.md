```bash

# namespace 생성
$ vault namespace create jenkins 

# KV secret 활성화
$ vault secrets enable -path=kv -ns=jenkins -version=2 kv
Success! Enabled the kv secrets engine at: kv/

# KV 데이터 생성
$ vault kv put -ns=jenkins kv/secret password=1234
= Secret Path =
kv/data/secret

======= Metadata =======
Key                Value
---                -----
created_time       2023-10-26T01:02:38.725834966Z
custom_metadata    <nil>
deletion_time      n/a
destroyed          false
version            1

# ssh otp policy 생성
$ vault policy write -ns=jenkins kv_jenkins_policy -<<EOF
path "kv/data/secret" {
  capabilities = ["read"]
}
EOF
Success! Uploaded policy: kv_jenkins_policy
```

# Jenkins Server

접속 주소 : http://43.201.98.46:8080/

계정 : admin / 1234qwer

## Vault Plugin 설치

jenkins 관리 - System Configuration - Plugins

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/f90702de-2a7d-47cd-9529-ae9583e2d8f1/40c5b9fe-dd81-45d0-bdec-e5e194d17361/Untitled.png)



![Untitled](https://github.com/jslim1995/insideinfo-test/assets/100335118/5839bd6d-d8ab-4018-80c1-d91d24ade73f)

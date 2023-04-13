# Vault PKI Secret Engine을 이용한 mTLS

Vault의 PKI Secret Engine을 활용하여 mTLS 구현

## 1. Vault 설정

### 1.1. Vault Dev 서버 실행

[Command]

```bash
vault server -dev -dev-root-token-id=root
```

* -`dev`: Vault Dev 모드로 실행
* `-dev-root-token-id`: Dev 모드에서만 적용. 지정한 문자열으로 Initial root token을 대체.

[Output]

```
WARNING! dev mode is enabled! In this mode, Vault runs entirely in-memory
and starts unsealed with a single unseal key. The root token is already
authenticated to the CLI, so you can immediately begin using Vault.

You may need to set the following environment variables:

    $ export VAULT_ADDR='http://127.0.0.1:8200'

The unseal key and root token are displayed below in case you want to
seal/unseal the Vault or re-authenticate.

Unseal Key: OFT...HE=
Root Token: root

Development mode should NOT be used in production installations!
```



### 1.2. Vault 환경변수 설정

[Command]

```bash
export VAULT_ADDR="http://127.0.0.1:8200"
```

```bash
vault login
```

```bash
Token (will be hidden): root
```

[Output]

```
Success! You are now authenticated. The token information displayed below
is already stored in the token helper. You do NOT need to run "vault login"
again. Future Vault requests will automatically use this token.

Key                  Value
---                  -----
token                root
token_accessor       x2Pvp47BRlhEFyK35hACDIze
token_duration       ∞
token_renewable      false
token_policies       ["root"]
identity_policies    []
policies             ["root"]
```



## 2. PKI 설정

### 2.1. PKI Secret Engine 활성화

[Command]

```bash
vault secrets enable pki
```

[Output]

```
Success! Enabled the pki secrets engine at: pki/
```



### 2.2. PKI Secret Engine TTL 변경

[Command]

```bash
vault secrets tune -max-lease-ttl=87600h pki
```

* Default TTL은 32일(768h)
* 10년(87600h) 으로 설정

[Output]

```
Success! Tuned the secrets engine at: pki/
```



### 2.3. Root CA 생성

[Command]

```bash
vault write pki/root/generate/internal \
    key_bits=2048 \
    private_key_format=pem \
    signature_bits=256 \
    country=KR \
    province=Seongnam-si \
    locality=Bundang-gu \
    organization=DDIM \
    ou=Engineer \
    common_name=example.com \
    ttl=87600h
```

[Output]

```
Key              Value
---              -----
certificate      -----BEGIN CERTIFICATE-----
...
-----END CERTIFICATE-----
expiration       1996496548
issuer_id        86b171c1-20fa-0641-0b9d-91f4eb4ae976
issuer_name      n/a
issuing_ca       -----BEGIN CERTIFICATE-----
...
-----END CERTIFICATE-----
key_id           f0146593-889e-107e-3214-b2a2eaea36c0
key_name         n/a
serial_number    70:24:6b:a9:3b:17:11:fb:1c:b2:36:a1:27:e9:5e:06:3a:ca:49:ba
```

* PKI Secret Engine  `pki` 경로에서 `role`을 생성하여 발급되는 인증서들은 모두 해당 Root CA에 종속됨



### 2.4. CRL 생성

Certificate Revocation List(인증서 **해지** 목록) 엔드포인트 작성.
클라이언트는 `CRL`의 `URL`에서 `CRL`을 다운로드 받아 인증서의 폐기여부를 확인

```mermaid
sequenceDiagram
    participant Client
    participant CA

    Client->>CA: 1. HTTP 또는 LDAP을 이용, 인증서를 통해 교부받은 URL을 통해 CRL을 요청
    CA->>Client: 2. 요청받은 CRL을 응답

    Note over Client,CA: Client는 CRL 정보를 파싱하여 현재 접속하려는 사이트의 인증서가 CRL에 포함되어 있는지 점검
    Note over Client,CA: CRL에는 만료 인증서에 대한 시리얼 번호가 포함
```

[Command]

```bash
vault write pki/config/urls \
    issuing_certificates="http://127.0.0.1:8200/v1/pki/ca" \
    crl_distribution_points="http://127.0.0.1:8200/v1/pki/crl"
```

[Output]

```
Success! Data written to: pki/config/urls
```



### 2.5. Role 생성

> https://developer.hashicorp.com/vault/api-docs/secret/pki#create-update-role

[Command]

```bash
vault write pki/roles/example-dot-com \
    allowed_domains="example.com" \
    allow_bare_domains=true \
    allowed_wildcard_certificates=true \
    allow_subdomains=true \
    max_ttl="720h"
```

[Output]

```
Success! Data written to: pki/roles/example-dot-com
```



### 2.6. 인증서 발급

[Command]

```bash
vault write pki/issue/example-dot-com \
    common_name=service-a.example.com
```

[Output]

```
Key                 Value
---                 -----
ca_chain            [-----BEGIN CERTIFICATE-----
...
-----END CERTIFICATE-----]
certificate         -----BEGIN CERTIFICATE-----
...
-----END CERTIFICATE-----
expiration          1683986821
issuing_ca          -----BEGIN CERTIFICATE-----
...
-----END CERTIFICATE-----
private_key         -----BEGIN RSA PRIVATE KEY-----
...
-----END RSA PRIVATE KEY-----
private_key_type    rsa
serial_number       1b:dc:6d:1e:a4:44:41:b6:ec:41:6b:a1:3d:43:56:6c:b0:11:5e:63
```

* `ca_chain`, `issuing_ca`: 2.3. 과정의 Root CA의 값과 동일(Root CA에 종속)



## 3. Vault Agent 설정

### 3.1. Vault Agent 정책 설정

[Command]

```bash
cd vault_agent
```

```bash
vault pollicy write pki pki_policy.hcl
```

[Output]

```
Success! Uploaded policy: pki
```



### 3.2. Vault Agent 용 approle 인증 설정

#### 3.2.1. Vault Agent 용 approle 활성화

[Command]

```bash
vault auth enable approle
```

[Output]

```
Success! Enabled approle auth method at: approle/
```



#### 3.2.2. Vault Agent 용 approle의 role 생성

[Command]

```bash
vault write auth/approle/role/pki-agent \
	secret_id_ttl=120m \
	token_ttl=60m \
	token_max_tll=120m \
	policies="pki"
```

[Output]

```bash
Success! Data written to: auth/approle/role/pki-agent
```



#### 3.2.3. RoleID 저장

[Command]

```bash
vault read -field=role_id auth/approle/role/pki-agent/role-id > roleid
```

* `-field`: Command의 Output 중 특정 항목만 출력

일반적인 RoleID 정보 출력

[Command]

```bash
vault read auth/approle/role/pki-agent/role-id
```

[Output]

```
Key        Value
---        -----
role_id    bc486c38-d7be-a685-4026-3b46e1ab634e
```



#### 3.2.4. SecretID 생성 및 저장

[Command]

```bash
vault write -f -field=secret_id auth/approle/role/pki-agent/secret-id > secretid
```

* `-field`: Command의 Output 중 특정 항목만 출력

일반적인 SecretID 생성 및 출력

[Command]

```bash
vault write -f auth/approle/role/pki-agent/secret-id
```

[Output]

```
Key                   Value
---                   -----
secret_id             2adb95a6-fa31-c248-9ab9-17bd152e7a03
secret_id_accessor    5d35140f-6158-8f2f-1af4-1f2481a17228
secret_id_num_uses    0
secret_id_ttl         2h
```

* Vault Agent 재기동 시 `secret_id` 재발급 필요



### 3.3. Template 확인 및 수정

Vault Agent는 Template을 사용하여 Secret을 특정 파일로 랜더링 가능

[vault_agent.hcl 파일 예시]

```hcl
template {
  source      = "ca-a.tpl"
  destination = "../cert/ca.crt"
}

template {
  source      = "cert-a.tpl"
  destination = "../cert/service-a.crt"
}

template {
  source      = "key-a.tpl"
  destination = "../cert/service-a.key"
}

template {
  source      = "cert-b.tpl"
  destination = "../cert/service-b.crt"
}

template {
  source      = "key-b.tpl"
  destination = "../cert/service-b.key"
}
```

* Template(`.tpl` 파일)에 대한 랜더링 결과를 특정 파일에 저장하도록 명시

[ca-a.tpl 파일 예시]

```tpl
# ca-a.tpl
{{- /* ca-a.tpl */ -}}
{{ with secret "pki/issue/example-dot-com" "common_name=service-a.example.com" "ttl=2m" }}
{{ .Data.issuing_ca }}{{ end }}
```

* `pki/issue/example-dot-com`에서 `common_name=service-a.example.com`인 인증서를 발급
* `.Data.issuing_ca`: Vault로부터 받는 결과 중 `issuing_ca` 값을 추출



[Command]

```bash
vi ca-a.tpl
```

```tpl
# ca-a.tpl
{{- /* ca-a.tpl */ -}}
{{ with secret "pki/issue/example-dot-com" "common_name=service-a.example.com" "ttl=10h" }}
{{ .Data.issuing_ca }}{{ end }}
```

* `ttl` 값 적당히 조절



### 3.4. Vault Agent 실행

[Command]

```bash
vault agent -config=vault_agent.hcl -log-level=debug
```

[Output]

```
2023-04-13T14:07:42.797+0900 [DEBUG] (runner) rendering "ca-a.tpl" => "../cert/ca.crt"
2023-04-13T14:07:42.797+0900 [DEBUG] (runner) checking template a04612e63b9a03a45ef968a8984a23db
2023-04-13T14:07:42.797+0900 [DEBUG] (runner) rendering "cert-a.tpl" => "../cert/service-a.crt"
2023-04-13T14:07:42.797+0900 [DEBUG] (runner) checking template 850589d81f7afe64c7c5a0a8440c8569
2023-04-13T14:07:42.797+0900 [DEBUG] (runner) rendering "key-a.tpl" => "../cert/service-a.key"
2023-04-13T14:07:42.797+0900 [DEBUG] (runner) checking template 60e7f2683d2c76a501eb54879bf89ad2
2023-04-13T14:07:42.797+0900 [DEBUG] (runner) rendering "cert-b.tpl" => "../cert/service-b.crt"
2023-04-13T14:07:42.801+0900 [INFO] (runner) rendered "cert-b.tpl" => "../cert/service-b.crt"
2023-04-13T14:07:42.801+0900 [DEBUG] (runner) checking template 1fb22b9f15857b7eeb0b68a3c9ac6d20
2023-04-13T14:07:42.802+0900 [DEBUG] (runner) rendering "key-b.tpl" => "../cert/service-b.key"
2023-04-13T14:07:42.807+0900 [INFO] (runner) rendered "key-b.tpl" => "../cert/service-b.key"
```

* Vault Agent가 새로 생성한 `ca.cert` 파일을 키체인에 새로 등록해 주어야 한다.



Browser에서 `https://service-a.example.com:7443/w-mtls` 접속 시 인증서가 제대로 동작하는 것을 알 수 있다.

![Screenshot 1](./img/Screenshot 1.png)

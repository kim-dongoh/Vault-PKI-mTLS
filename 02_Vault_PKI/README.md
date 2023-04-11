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



### 1.3. PKI 설정

#### 1.3.1. PKI Secret Engine 활성화

[Command]

```bash
vault secrets enable pki
```

[Output]

```
Success! Enabled the pki secrets engine at: pki/
```



#### 1.3.2. PKI Secret Engine TTL 변경

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



#### 1.3.3. Root CA 생성

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



#### 1.3.4. CRL 생성

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



#### 1.3.5. Role 생성

[Command]

```bash
vault write pki/roles/example-dot-com \
    allowed_domains="example.com" \
    allow_subdomains=true \
    max_ttl="720h"
```

[Output]

```
Success! Data written to: pki/roles/example-dot-com
```



#### 1.3.6. 발급

```bash
vault write pki/issue/example-dot-com \
    common_name=service-a.example.com
```

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


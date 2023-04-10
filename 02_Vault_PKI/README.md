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
vault secrets tune -max-lease-ttl=8760h pki
```

* Default TTL은 32일(768h)
* 1년(8760h) 으로 설정

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



#### 1.3.4. CRL 생성




# vaultrs

<p align="center">
    <a href="https://crates.io/crates/vaultrs">
        <img src="https://img.shields.io/crates/v/vaultrs">
    </a>
    <a href="https://docs.rs/vaultrs">
        <img src="https://img.shields.io/docsrs/vaultrs" />
    </a>
    <a href="https://www.vaultproject.io/">
        <img src="https://img.shields.io/badge/Vault-1.8.2-green" />
    </a>
    <a href="https://github.com/jmgilman/vaultrs/actions/workflows/ci.yml">
        <img src="https://github.com/jmgilman/vaultrs/actions/workflows/ci.yml/badge.svg"/>
    </a>
</p>

> An asynchronous Rust client library for the [Hashicorp Vault][1] API

The following features are currently supported:

- Auth
  - [AppRole](https://www.vaultproject.io/docs/auth/approle)
  - [AWS](https://www.vaultproject.io/docs/auth/aws)
  - [JWT/OIDC](https://www.vaultproject.io/api-docs/auth/jwt)
  - [Kubernetes](https://www.vaultproject.io/docs/auth/kubernetes)
  - [Token](https://www.vaultproject.io/docs/auth/token)
  - [Userpass](https://www.vaultproject.io/docs/auth/userpass)
- Secrets
  - [AWS](https://developer.hashicorp.com/vault/docs/secrets/aws)
  - [Databases](https://www.vaultproject.io/api-docs/secret/databases)
  - [KV v1](https://www.vaultproject.io/docs/secrets/kv/kv-v1)
  - [KV v2](https://www.vaultproject.io/docs/secrets/kv/kv-v2)
  - [PKI](https://www.vaultproject.io/docs/secrets/pki)
  - [SSH](https://www.vaultproject.io/docs/secrets/ssh)
  - [Transit](https://www.vaultproject.io/api-docs/secret/transit)
- Sys
  - [Health](https://www.vaultproject.io/api-docs/system/health)
  - [Policies](https://www.vaultproject.io/api-docs/system/policy)
  - [Sealing](https://www.vaultproject.io/api-docs/system/seal)
  - [Wrapping](https://www.vaultproject.io/docs/concepts/response-wrapping)

See something missing?
[Open an issue](https://github.com/jmgilman/vaultrs/issues/new).

## Installation

First, choose one of the two TLS implementations for `vaultrs`' connection to
Vault:

- `rustls` (default) to use [Rustls](https://github.com/rustls/rustls)
- `native-tls` to use
  [rust-native-tls](https://github.com/sfackler/rust-native-tls), which builds
  on your platform-specific TLS implementation.

Then, add `vaultrs` as a dependency to your cargo.toml:

1. To use [Rustls](https://github.com/rustls/rustls), import as follows:

```toml
[dependencies]
vaultrs = "0.7.0"
```

2. To use [rust-native-tls](https://github.com/sfackler/rust-native-tls), which
   builds on your platform-specific TLS implementation, specify:

```toml
[dependencies]
vaultrs = { version = "0.6.2", default-features = false, features = [ "native-tls" ] }
```

## Usage

### Basic

The client is used to configure the connection to Vault and is required to be
passed to all API calls for execution. Behind the scenes it uses an asynchronous
client from [Reqwest](https://docs.rs/reqwest/) for communicating to Vault.

```rust
use vaultrs::client::{VaultClient, VaultClientSettingsBuilder};

// Create a client
let client = VaultClient::new(
    VaultClientSettingsBuilder::default()
        .address("https://127.0.0.1:8200")
        .token("TOKEN")
        .build()
        .unwrap()
).unwrap();
```

### Secrets

#### AWS

The library currently supports all operations available for the
AWS Secret Engine.

See [tests/aws.rs](./tests/aws.rs) for more examples.

```rust
// Mount AWS SE
server.mount_secret(client, path, "aws").await?;
let endpoint = AwsSecretEngineEndpoint { path: path }

// Configure AWS SE
aws::config::set(client, &endpoint.path, "access_key", "secret_key", Some(SetConfigurationRequest::builder()        
    .max_retries(3)
    .region("eu-central-1")
)).await?,

// Create HVault role
aws::roles::create_update(client, &endpoint.path, "my_role", "assumed_role", Some(CreateUpdateRoleRequest::builder()
        .role_arns( vec!["arn:aws:iam::123456789012:role/test_role"] )
)).await?

// Generate credentials
let res = aws::roles::credentials(client, &endpoint.path, "my_role", Some(GenerateCredentialsRequest::builder()
    .ttl("3h")
)).await?;

let creds = res.unwrap();
// creds.access_key
// creds.secret_key
// creds.security_token
```

#### Key Value v2

The library currently supports all operations available for version 2 of the
key/value store.

```rust
use serde::{Deserialize, Serialize};
use vaultrs::kv2;

// Create and read secrets
#[derive(Debug, Deserialize, Serialize)]
struct MySecret {
    key: String,
    password: String,
}

let secret = MySecret {
    key: "super".to_string(),
    password: "secret".to_string(),
};
kv2::set(
    &client,
    "secret",
    "mysecret",
    &secret,
).await;

let secret: MySecret = kv2::read(&client, "secret", "mysecret").await.unwrap();
println!("{}", secret.password); // "secret"
```

#### Key Value v1

The library currently supports all operations available for version 1 of the
key/value store.

```rust
let my_secrets = HashMap::from([
    ("key1".to_string(), "value1".to_string()),
    ("key2".to_string(), "value2".to_string())
]);

kv1::set(&client, mount, "my/secrets", &my_secrets).await.unwrap();

let read_secrets: HashMap<String, String> = kv1::get(&client, &mount, "my/secrets").await.unwrap();

println!("{:}", read_secrets.get("key1").unwrap()); // value1

let list_secret = kv1::list(&client, &mount, "my").await.unwrap();

println!("{:?}", list_secret.data.keys); // [ "secrets" ]

kv1::delete(&client, &mount, "my/secrets").await.unwrap();
```

### PKI

The library currently supports all operations available for the PKI secrets
engine.

```rust
use vaultrs::api::pki::requests::GenerateCertificateRequest;
use vaultrs::pki::cert;

// Generate a certificate using the PKI backend
let cert = cert::generate(
    &client,
    "pki",
    "my_role",
    Some(GenerateCertificateRequest::builder().common_name("test.com")),
).await.unwrap();
println!("{}", cert.certificate) // "{PEM encoded certificate}"
```

### Transit

The library supports most operations for the
[Transit](https://www.vaultproject.io/api-docs/secret/transit) secrets engine,
other than importing keys or `batch_input` parameters.

```rust
use vaultrs::api::transit::requests::CreateKeyRequest;
use vaultrs::api::transit::KeyType;

// Create an encryption key using the /transit backend
key::create(
    &client,
    "transit",
    "my-transit-key",
    Some(CreateKeyRequest::builder()
       .derive(true)
       .key_type(KeyType::Aes256Gcm96)
       .auto_rotate_period("30d")),
).await.unwrap();
```

### Wrapping

All requests implement the ability to be
[wrapped](https://www.vaultproject.io/docs/concepts/response-wrapping). These
can be passed in your application internally before being unwrapped.

```rust
use vaultrs::api::ResponseWrapper;
use vaultrs::api::sys::requests::ListMountsRequest;

let endpoint = ListMountsRequest::builder().build().unwrap();
let wrap_resp = endpoint.wrap(&client).await; // Wrapped response
assert!(wrap_resp.is_ok());

let wrap_resp = wrap_resp.unwrap(); // Unwrap Result<>
let info = wrap_resp.lookup(&client).await; // Check status of this wrapped response
assert!(info.is_ok());

let unwrap_resp = wrap_resp.unwrap(&client).await; // Unwrap the response
assert!(unwrap_resp.is_ok());

let info = wrap_resp.lookup(&client).await; // Error: response already unwrapped
assert!(info.is_err());
```

## Error Handling and Tracing

All errors generated by this crate are wrapped in the `ClientError` enum
provided by the crate. API warnings are automatically captured via `tracing` and
API errors are captured and returned as their own variant. Connection related
errors from `rustify` are wrapped and returned as a single variant.

All top level API operations are instrumented with `tracing`'s `#[instrument]`
attribute.

## Testing

See the the [tests](tests) directory for tests. Run tests with `cargo test`.

**Note**: All tests rely on bringing up a local Vault development server using
Docker. In order to run tests Docker must be running locally (Docker Desktop
works).

## Contributing

Check out the [issues][2] for items needing attention or submit your own and
then:

1. Fork the repo (<https://github.com/jmgilman/vaultrs/fork>)
2. Create your feature branch (git checkout -b feature/fooBar)
3. Commit your changes (git commit -am 'Add some fooBar')
4. Push to the branch (git push origin feature/fooBar)
5. Create a new Pull Request

See [CONTRIBUTING](CONTRIBUTING.md) for extensive documentation on the
architecture of this library and how to add additional functionality to it.

[1]: https://www.vaultproject.io/
[2]: https://github.com/jmgilman/vaultrs/issues

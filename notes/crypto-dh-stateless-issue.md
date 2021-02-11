### Node.js test-crypto-dh-stateless.js
This document contains notes around an issue with this test that was discovered
when upgrading Node.js to OpenSSL 3.0 (currently only dynamically linking to
the latest OpenSSL uptream master).

### Error

```console
$ out/Debug/node /home/danielbevenius/work/nodejs/openssl/test/parallel/test-crypto-dh-stateless.js
node:assert:580
      throw err;
      ^

AssertionError [ERR_ASSERTION]: Expected values to be strictly deep-equal:
+ actual - expected

  Comparison {
+   code: 'ERR_OSSL_DH_INVALID_PUBLIC_KEY',
-   code: 'ERR_OSSL_EVP_DIFFERENT_PARAMETERS',
    name: 'Error'
  }
    at Object.<anonymous> (/home/danielbevenius/work/nodejs/openssl/test/parallel/test-crypto-dh-stateless.js:157:10)
    at Module._compile (node:internal/modules/cjs/loader:1091:14)
    at Object.Module._extensions..js (node:internal/modules/cjs/loader:1120:10)
    at Module.load (node:internal/modules/cjs/loader:971:32)
    at Function.Module._load (node:internal/modules/cjs/loader:812:14)
    at Function.executeUserEntryPoint [as runMain] (node:internal/modules/run_main:76:12)
    at node:internal/main/run_main_module:17:47 {
  generatedMessage: true,
  code: 'ERR_ASSERTION',
  actual: Error: error:02800066:Diffie-Hellman routines::invalid public key
      at Object.diffieHellman (node:internal/crypto/diffiehellman:317:10)
      at test (/home/danielbevenius/work/nodejs/openssl/test/parallel/test-crypto-dh-stateless.js:32:23)
      at assert.throws.name (/home/danielbevenius/work/nodejs/openssl/test/parallel/test-crypto-dh-stateless.js:158:5)
      at getActual (node:assert:701:5)
      at Function.throws (node:assert:841:24)
      at Object.<anonymous> (/home/danielbevenius/work/nodejs/openssl/test/parallel/test-crypto-dh-stateless.js:157:10)
      at Module._compile (node:internal/modules/cjs/loader:1091:14)
      at Object.Module._extensions..js (node:internal/modules/cjs/loader:1120:10)
      at Module.load (node:internal/modules/cjs/loader:971:32)
      at Function.Module._load (node:internal/modules/cjs/loader:812:14) {
    library: 'Diffie-Hellman routines',
    reason: 'invalid public key',
    code: 'ERR_OSSL_DH_INVALID_PUBLIC_KEY'
  },
  expected: { name: 'Error', code: 'ERR_OSSL_EVP_DIFFERENT_PARAMETERS' },
  operator: 'throws'
}
```
The error `ERR_OSSL_DH_INVALID_PUBLIC_KEY` is defined in Node.js

crypto/err/openssl.txt
```
DH_R_INVALID_PUBKEY:102:invalid public key
```
include/openssl/dherr.h:
```c
  define DH_R_INVALID_PUBKEY                              102
```
crypto/dh/dh_err.c:
```c
{ERR_PACK(ERR_LIB_DH, 0, DH_R_INVALID_PUBKEY), "invalid public key"},
```
Lets set a break point and see which one of the locations that raise this error
gets hit:
```console
(lldb) br s -f dh_key.c -l 79
```
Below is the backtrace:
```
(lldb) bt 
* thread #1, name = 'node', stop reason = breakpoint 3.1
  * frame #0: 0x00007ffff7c7a499 libcrypto.so.3`compute_key(key="\x80\r\xb4\x05", pub_key=0x0000000005b45560, dh=0x0000000005b40110) at dh_key.c:79:9
    frame #1: 0x00007ffff7c7a601 libcrypto.so.3`DH_compute_key(key="\x80\r\xb4\x05", pub_key=0x0000000005b45560, dh=0x0000000005b40110) at dh_key.c:110:11
    frame #2: 0x00007ffff7e5e729 libcrypto.so.3`dh_plain_derive(vpdhctx=0x00000000059411f0, secret="\x80\r\xb4\x05", secretlen=0x00007fffffffba60, outlen=192) at dh_exch.c:149:15
    frame #3: 0x00007ffff7e5e95c libcrypto.so.3`dh_derive(vpdhctx=0x00000000059411f0, secret="\x80\r\xb4\x05", psecretlen=0x00007fffffffba60, outlen=192) at dh_exch.c:209:20
    frame #4: 0x00007ffff7d21ef4 libcrypto.so.3`EVP_PKEY_derive(ctx=0x000000000597aa50, key="\x80\r\xb4\x05", pkeylen=0x00007fffffffba60) at exchange.c:429:11
    frame #5: 0x0000000001239329 node`node::crypto::(anonymous namespace)::StatelessDiffieHellman(env=0x00000000059751d0, our_key=ManagedEVPPKey @ 0x00007fffffffbb50, their_key=ManagedEVPPKey @ 0x00007fffffffbb30) at crypto_dh.cc:554:22
    frame #6: 0x0000000001239817 node`node::crypto::DiffieHellman::Stateless(args=0x00007fffffffbbf0) at crypto_dh.cc:610:71
```

In src/crypto/crypto_dh.cc we have the function `Stateless`:
```c++
void DiffieHellman::Stateless(const FunctionCallbackInfo<Value>& args) {
  Environment* env = Environment::GetCurrent(args);

  CHECK(args[0]->IsObject() && args[1]->IsObject());
  KeyObjectHandle* our_key_object;
  ASSIGN_OR_RETURN_UNWRAP(&our_key_object, args[0].As<Object>());
  CHECK_EQ(our_key_object->Data()->GetKeyType(), kKeyTypePrivate);
  KeyObjectHandle* their_key_object;
  ASSIGN_OR_RETURN_UNWRAP(&their_key_object, args[1].As<Object>());
  CHECK_NE(their_key_object->Data()->GetKeyType(), kKeyTypeSecret);

  ManagedEVPPKey our_key = our_key_object->Data()->GetAsymmetricKey();
  ManagedEVPPKey their_key = their_key_object->Data()->GetAsymmetricKey();

  AllocatedBuffer out = StatelessDiffieHellman(env, our_key, their_key);
  if (out.size() == 0)
    return ThrowCryptoError(env, ERR_get_error(), "diffieHellman failed");

  args.GetReturnValue().Set(out.ToBuffer().FromMaybe(Local<Value>()));
}

namespace {
AllocatedBuffer StatelessDiffieHellman(
    Environment* env,
    ManagedEVPPKey our_key,
    ManagedEVPPKey their_key) {
  size_t out_size;

  EVPKeyCtxPointer ctx(EVP_PKEY_CTX_new(our_key.get(), nullptr));
  if (!ctx ||
      EVP_PKEY_derive_init(ctx.get()) <= 0 ||
      EVP_PKEY_derive_set_peer(ctx.get(), their_key.get()) <= 0 ||
      EVP_PKEY_derive(ctx.get(), nullptr, &out_size) <= 0)
    return AllocatedBuffer();

  AllocatedBuffer result = AllocatedBuffer::AllocateManaged(env, out_size);
  CHECK_NOT_NULL(result.data());

  unsigned char* data = reinterpret_cast<unsigned char*>(result.data());
  if (EVP_PKEY_derive(ctx.get(), data, &out_size) <= 0)
    return AllocatedBuffer();

  ZeroPadDiffieHellmanSecret(out_size, &result);
  return result;

void DiffieHellman::Initialize(Environment* env, Local<Object> target) {
  ...

  env->SetMethodNoSideEffect(target, "statelessDH", DiffieHellman::Stateless);
  ...
}
```
And we can see this being imported in `lib/internal/crypto/diffiehellman.js`:
```js
const {
  ...
  statelessDH,
  ...
} = internalBinding('crypto');

function diffieHellman(options) {
  validateObject(options, 'options');

  const { privateKey, publicKey } = options;
  ...

  const privateType = privateKey.asymmetricKeyType;
  const publicType = publicKey.asymmetricKeyType;
  if (privateType !== publicType || !dhEnabledKeyTypes.has(privateType)) {
    throw new ERR_CRYPTO_INCOMPATIBLE_KEY('key types for Diffie-Hellman',
                                          `${privateType} and ${publicType}`);
  }

  return statelessDH(privateKey[kHandle], publicKey[kHandle]);
```

In ossl_ffc_validate_public_key_partial we can find the following:
```c
    if (BN_cmp(pub_key, tmp) >= 0) {
        *ret |= FFC_ERROR_PUBKEY_TOO_LARGE;
        goto err;
    }
```
This is where our error originates from this finite field cryptography (ffc)
public key validation.
```console
(lldb) br s -f ffc_key_validate.c -l 46
```
And the the following code will raise the error we see in the test:
```c
if (!DH_check_pub_key(dh, pub_key, &check_result) || check_result) {
   ERR_raise(ERR_LIB_DH, DH_R_INVALID_PUBKEY);
   goto err;
}

```

__WIP__
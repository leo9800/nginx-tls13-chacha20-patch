## What does this patch do?

1. Enabling nginx to set the sequence of TLS 1.3 cipher-suites.
2. Prefer to use CHACHA20 ciphers on those devices with no AES instructions.

## Compatibility

 - In general, patch nginx code with the patch file with corressponding version, e.g. version suffix `_1_15_9` is for nginx 1.15.9
 - The patch files with suffix `_1_15_8` were designed for nginx 1.15.6 to 1.15.8

## How to use?

Patch and compile nginx:

```
$ cd {nginx_source_path}
$ patch -p1 < {path_to_patch}/nginx_tls13_chacha20_{NGINX_VERSION}.patch
$ ./configure --with-openssl={/path/to/openssl-1.1.1} {your_arguments}
$ make
$ sudo make install
```
Tips: use `/path/to/nginx -V` to get the compile arguments from the installed nginx, and replace `{your_arguments}` with them.

Modify nginx config file:

```
http {
# ...
ssl_protocols TLSv1.2 TLSv1.3;
ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE-RSA-CHACHA20-POLY1305;
ssl_ciphers_tls13 TLS_AES_128_GCM_SHA256:TLS_CHACHA20_POLY1305_SHA256;
ssl_prefer_server_ciphers on;
ssl_prefer_chacha20 on;
# ...
}
```

If you applied the TLS 1.3-only patch (`nginx_tls13.patch`), do not add `ssl_prefer_chacha20 on;` or `nginx -t` will fail and nginx won't startup.

## How does it work?

TL;DR

### TLS 1.3 cipher-suites

OpenSSL has introduced a new function `int SSL_CTX_set_ciphersuites(SSL_CTX *ctx, const char *str);` for setting cipher-suites for TLS 1.3, no longer use the legacy function for TLS 1.2(`int SSL_CTX_set_cipher_list(SSL_CTX *ctx, const char *str);`) and below.

Some new functions has been added to the nginx code with this patch to set TLS 1.3 cipher-suites with the newly-introduced OpenSSL function.

### CHACHA20 priority

A new parameter, `SSL_OP_PRIORITIZE_CHACHA`, was introduced to the alpha version of OpenSSL 1.1.1's function `long SSL_CTX_set_options(SSL_CTX *ctx, long options);`, if this parameter applies, OpenSSL will prefer to use CHACHA20 cipher-suites when connecting with a device without AES instructions.

SSL library may set the CHACHA20 cipher-suites as the first options when running on devices without AES instruction, for example:

Chrome Cipher-suites on an Intel core i3-3220[W/O AES-NI]:

```
TLS_CHACHA20_POLY1305_SHA256:TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:{other_ciphers}
```

Chrome Cipher-suites on an Intel core i7-3770[W/ AES-NI]:

```
TLS_AES_128_GCM_SHA256:TLS_CHACHA20_POLY1305_SHA256:TLS_AES_256_GCM_SHA384:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:{other_ciphers}
```
Whether CHACHA20 suites as the first option of client is the judgement.

## Requirements

TLS 1.3 requires OpenSSL 1.1.1 or a higher version, or other OpenSSL-compatible SSL implements with TLS 1.3 support.

CHACHA20 periority relies on OpenSSL 1.1.1+ or [patched](https://github.com/Hardrain980/openssl-1.1.0-patch) OpenSSL 1.1.0 

Using patched OpenSSL 1.1.0 branch is _NOT_ recommended, as it has reached its EOL (end of life), security fixes may not available anymore, you should upgrade to OpenSSL 1.1.1

## Test result

~~This patch has been tested with nginx 1.15.6 and OpenSSL 1.1.1, it passed both compilation and run-time tests.~~

\[20190303\] Update: This patch has been tested with nginx 1.15.9 and OpenSSL 1.1.1b, it passed both compilation and run-time tests.

\[20190525\] Update: This patch has been tested with nginx 1.17.0 and OpenSSL 1.1.1b, it passed both compilation and run-time tests.

### Informations (may helpful)

TLS 1.3 cipher-suites in OpenSSL-styled name:

```
TLS_AES_128_GCM_SHA256
TLS_AES_256_GCM_SHA384
TLS_CHACHA20_POLY1305_SHA256
TLS_AES_128_CCM_SHA256
TLS_AES_128_CCM_8_SHA256
```

### Reference:

- [/docs/manmaster/man3/SSL_CTX_set_options.html](https://www.openssl.org/docs/manmaster/man3/SSL_CTX_set_options.html)
- [/docs/manmaster/man3/SSL_CTX_set_cipher_list.html](https://www.openssl.org/docs/manmaster/man3/SSL_CTX_set_cipher_list.html)
- [patch/nginx_auto_using_PRIORITIZE_CHACHA.patch at master · kn007/patch](https://github.com/kn007/patch/blob/master/nginx_auto_using_PRIORITIZE_CHACHA.patch)

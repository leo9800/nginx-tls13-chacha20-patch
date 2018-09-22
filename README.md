## What is this patch?

This patch enabled nginx to set the sequence of TLSv1.3 cipher-suites when compiled with OpenSSL 1.1.1+

## Why this patch was made?

OpenSSL has introduced a new function `int SSL_CTX_set_ciphersuites(SSL_CTX *ctx, const char *str);` for defining the sequence of TLSv1.3 cipher-suites, no longer using the legacy way already implemented for TLSv1.2 (or legacy SSLv3 to TLSv1.1).

OpenSSL has decleared that, the legacy function (`int SSL_CTX_set_cipher_list(SSL_CTX *ctx, const char *str);`) is only for legacy protocols (SSLv3 to TLSv1.2), and the TLSv1.3 implement use the _exclusively_ use the new function (`int SSL_CTX_set_ciphersuites(SSL_CTX *ctx, const char *str);`) to set the sequence of cipher-suites.

Nginx's `ssl_ciphers` parameter uses `int SSL_CTX_set_cipher_list(SSL_CTX *ctx, const char *str);` to set cipher-suites, it is unable to customize TLSv1.3 cipher-suites with the original code.

## How this patch works?

This patch add some functions to parse a new parameter, `ssl_ciphers_tls13`, in the configure file, get the TLSv1.3 cipher-suites string and pass it to OpenSSL with `int SSL_CTX_set_ciphersuites(SSL_CTX *ctx, const char *str);`.

```shell
$ cd {nginx_source_path}
$ patch -p1 < {path_to_patch}/nginx_tls13.patch
$ ./configure --with-openssl={/path/to/openssl-1.1.1} {your_arguments}
$ make
$ sudo make install
```

```
server {
  listen 443 ssl http2;
  server_name domain.tld;
  // ...
  ssl_protocols TLSv1.2 TLSv1.3;
  ssl_ciphers_tls13 TLS_AES_128_GCM_SHA256;
}
```
the parameters can be:

1. TLS_AES_128_GCM_SHA256
2. TLS_AES_256_GCM_SHA384
3. TLS_CHACHA20_POLY1305_SHA256
4. TLS_AES_128_CCM_SHA256
5. TLS_AES_128_CCM_8_SHA256

the recommended string is `TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256`.

## Is it tested or has any bug?

Yes, it has passed the compilation and runtime tests with nginx 1.15.3 (latest@2018-09-17) and OpenSSL 1.1.1.

### Bug:

Due to unknown reason, you _must_ define `ssl_ciphers_tls13` parameter in every `server {}` blocks which listening as SSL, although TLSv1.3 is not enabled, or 'segmentation fault' will block the server from running.

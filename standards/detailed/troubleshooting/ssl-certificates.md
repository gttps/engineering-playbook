# SSL Certificate Troubleshooting

Check a public endpoint's certificate, expiry, and served chain:

```bash
openssl s_client -connect example.com:443 -servername example.com -showcerts </dev/null
```

-showcerts` prints every certificate in the chain (leaf → intermediates), each as a PEM block.
- `-servername` sets SNI, important for hosts serving multiple certs.
- `</dev/null` closes stdin so the command doesn’t hang waiting for input.

## Verify the chain is complete and valid
`s_client` will also report the verification result. Look for this line in the output:
```Verify return code: 0 (ok)```
A non-zero code (e.g. `21 unable to verify the first certificate`) usually means the server isn’t sending its intermediates.


# Check chain in a local file
```bash
# Verify a cert against a known chain/CA bundle
openssl verify -CAfile chain.pem -untrusted intermediates.pem cert.pem

# Split and inspect each cert in a bundle
openssl crl2pkcs7 -nocrl -certfile fullchain.pem | openssl pkcs7 -print_certs -noout
```

Add `-verify_hostname example.com` to also validate the hostname matches the cert.

- Pipe through `grep` for a fast subject/issuer readout:
```bash
openssl s_client -connect example.com:443 -servername example.com -showcerts </dev/null 2>/dev/null | grep -E "s:|i:"

```


# Online checkers and chain tools:

- [KeyCDN SSL tools](https://www.keycdn.com/ssl-tools)
- [Qualys SSL Labs Server Test](https://www.ssllabs.com/ssltest/)
- [WhatsMyChainCert](https://whatsmychaincert.com/) for checking and downloading certificate chains

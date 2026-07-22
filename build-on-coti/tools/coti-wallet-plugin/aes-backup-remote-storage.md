# Secure remote AES backup storage

{% hint style="warning" %}
**Not supported.** Remote / custom AES backup backends are **not** a supported or maintained product path for the COTI Wallet Plugin. The only supported encrypted-backup storage is browser **localStorage** (`onboardingServices.mode: 'localStorage'`). See [AES backup security model](aes-backup-security.md).
{% endhint %}

This page is retained only as historical design notes. Do not build production integrations against a remote AES backup API for this plugin. Do not prioritize security work on remote backup auth for this product.

## Supported alternative

```ts
configureCotiPlugin({
  onboardingServices: { mode: 'localStorage' },
});
```

## Related docs

* [AES backup security model](aes-backup-security.md)
* [AES Key Onboarding](aes-key-onboarding.md)
* [Configuration](configuration.md)

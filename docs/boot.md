# Android Booting Shenanigans

## Terminologies

- **rootdir**: the root directory (`/`). All files/folders/filesystems are stored in or mounted under rootdir. On Android, the filesystem may be either `rootfs` or the `system` partition.
- **`initramfs`**: a section in Android's boot image that the Linux kernel will use as `rootfs`. People also use the term **ramdisk** interchangeably
- **`recovery` and `boot` partition**: these 2 are actually very similar: both are Android boot images containing ramdisk and Linux kernel (plus some other stuff). The only difference is that booting `boot` partition will bring us to Android, while `recovery` has a minimalist self contained Linux environment for repairing and upgrading the device.
- **SAR**: System-as-root. That is, the device uses `system` as rootdir instead of `rootfs`
- **A/B, A-only**: For devices supporting [Seamless System Updates](https://source.android.com/devices/tech/ota/ab), it will have 2 slots of all read-only partitions; we call these **A/B devices**. To differentiate, non A/B devices will be called **A-only**
- **2SI**: Two Stage Init. The way Android 10+ boots. More info later.

Here are a few parameters to more precisely define a device's Android version:

- **LV**: Launch Version. The Android version the device is **launched** with. That is, the Android version pre-installed when the device first hit the market.
- **RV**: Running Version. The Android version the device is currently running on.

We will use **Android API level** to represent LV and RV. The mapping between API level and Android versions can be seen in [this table](https://source.android.com/setup/start/build-numbers#platform-code-names-versions-api-levels-and-ndk-releases). For example: Pixel XL is released with Android 7.1, and is running Android 10, these parameters will be `(LV = 25, RV = 29)`

## Boot Methods

Android booting can be roughly categorized into 3 major different methods. We provide a general rule of thumb to determine which method your device is most likely using, with exceptions listed separately.

Method | Initial rootdir | Final rootdir
:---: | --- | ---
**A** | `rootfs` | `rootfs`
**B** | `system` | `system`
**C** | `rootfs` | `system`

- **Method A - Legacy ramdisk**: This is how *all* Android devices used to boot (good old days). The kernel uses `initramfs` as rootdir, and exec `/init` to boot.
	- Devices that does not fall in any of Method B and C's criteria
- **Method B - Legacy SAR**: This method was first seen on Pixel 1. The kernel directly mounts the `system` partition as rootdir and exec `/init` to boot.
	- Devices with `(LV = 28)`
	- Google: Pixel 1 and 2. Pixel 3 and 3a when `(RV = 28)`.
	- OnePlus: 6 - 7
	- Maybe some `(LV < 29)` Android Go devices?
- **Method C - 2SI ramdisk SAR**: This method was first seen on Pixel 3 Android 10 developer preview. The kernel uses `initramfs` as rootdir and exec `/init` in `rootfs`. This `init` is responsible to mount the `system` partition and use it as the new rootdir, then finally exec `/system/bin/init` to boot.
	- Devices with `(LV >= 29)`
	- Devices with `(LV < 28, RV >= 29)`, excluding those that were already using Method B
	- Google: Pixel 3 and 3a with `(RV >= 29)`

### Discussion

From documents online, Google's definition of SAR only considers how the kernel boots the device (**Initial rootdir** in the table above), meaning that only devices using **Method B** is *officially* considered an SAR device from Google's standpoint.

However for Magisk, the real difference lies in what the device ends up using when fully booted (**Final rootdir** in the table above), meaning that **as far as Magisk's concern, both Method B and C is a form of SAR**, but just implemented differently. Every instance of SAR later mentioned in this document will refer to **Magisk's definition** unless specifically says otherwise.

The criteria for Method C is a little complicated, in layman's words: either your device is modern enough to launch with Android 10+, or you are running an Android 10+ custom ROM on a device that was using Method A.

- Any Method A device running Android 10+ will automatically be using Method C
- **Method B devices are stuck with Method B**, with the only exception being Pixel 3 and 3a, which Google retrofitted the device to adapt the new method.

SAR is a very important part of [Project Treble](https://source.android.com/devices/architecture#hidl) as rootdir should be tied to the platform. This is also the reason why Method B and C comes with `(LV >= ver)` criterion as Google has enforced all OEMs to comply with updated requirements every year.

## Some History

When Google released the first generation Pixel, it also introduced [A/B (Seamless) System Updates](https://source.android.com/devices/tech/ota/ab). Due to [storage size concerns](https://source.android.com/devices/tech/ota/ab/ab_faqs), there are several differences compared to A-only, the most relevant one being the removal of `recovery` partition and the recovery ramdisk being merged into `boot`.

Let's go back in time when Google is first designing A/B. If using SAR (only Boot Method B exists at that time), the kernel doesn't need `initramfs` to boot Android (because rootdir is in `system`). This mean we can be smart and just stuff the recovery ramdisk (containing the minimalist Linux environment) into `boot`, remove `recovery`, and let the kernel pick whichever rootdir to use (ramdisk or `system`) based on information from the bootloader.

As time passed from Android 7.1 to Android 10, Google introduced [Dynamic Partitions](https://source.android.com/devices/tech/ota/dynamic_partitions/implement). This is bad news for SAR, because the Linux kernel cannot directly understand this new partition format, thus unable to directly mount `system` as rootdir. This is when they came up with Boot Method C: always boot into `initramfs`, and let userspace handle the rest of booting. This includes deciding whether to boot into Android or recovery, or as they officially call: `USES_RECOVERY_AS_BOOT`.

Some modern devices using A/B with 2SI also comes with `recovery_a/_b` partitions. This is officially supported with Google's standard. These devices will then only use the boot ramdisk to boot into Android as recovery is stored on a separate partition.

## Piecing Things Together

With all the knowledge above, now we can categorize all Android devices into these different types:

Type | Boot Method | Partition | 2SI | Ramdisk in `boot`
:---: | :---: | :---: | :---: | :---:
**I** | A | A-only | No | `boot` ramdisk
**II** | B | A/B | Any | `recovery` ramdisk
**III** | B | A-only | Any | ***N/A***
**IV** | C | Any | Yes | Hybrid ramdisk

These types are ordered chronologically by the time they were first available.

- **Type I**: Good old legacy ramdisk boot
- **Type II**: Legacy A/B devices. Pixel 1 is the first device of this type, being both the first A/B and SAR device
- **Type III**: Late 2018 - 2019 devices that are A-only. **The worst type of device to ever exist as far as Magisk is concerned.**
- **Type IV**: All devices using Boot Method C are Type IV. A/B Type IV ramdisk can boot into either Android or recovery based on info from bootloader; A-only Type IV ramdisk can only boot into Android.

Further details on Type III devices: Magisk is always installed in the ramdisk of a boot image. For all other device types, because their `boot` partition have ramdisk included, Magisk can be easily installed by patching boot image through the Magisk app or flash zip in custom recovery. However for Type III devices, they are **limited to install Magisk into the `recovery` partition**. Magisk will not function when booted normally; instead Type III device owners have to always reboot to recovery to maintain Magisk access.

Some Type III devices' bootloader will still accept and provide `initramfs` that was manually added to the `boot` image to the kernel (e.g. some Xiaomi phones), but many device don't (e.g. Samsung S10, Note 10). It solely depends on how the OEM implements its bootloader.

https://cloud.google.com/logging/docs/view/query-library?hl=es_419&_gl=1*hfj1ev*_ga*ODAxNDI1ODg1LjE3NDc5MjA0ODQ.*_ga_WH2QY8WWF5*czE3NDc5Njk4MzQkbzMkZzAkdDE3NDc5Njk4NDAkajU0JGwwJGgwJGRLblNKTktqakZ1d1c3bmdJTms5YmFsVU9sUkJWSHJEMVhR#security-filters

protoPayload.methodName="google.api.serviceusage.v1.ServiceUsage.EnableService"

{
  "key": "enter",
  "command": "type",
  "args": { "text": "Hello World" },
  "when": "editorTextFocus"
}
https://cloud.google.com/logging/docs/view/query-library?hl=es_419&_gl=1*hfj1ev*_ga*ODAxNDI1ODg1LjE3NDc5MjA0ODQ.*_ga_WH2QY8WWF5*czE3NDc5Njk4MzQkbzMkZzAkdDE3NDc5Njk4NDAkajU0JGwwJGgwJGRLblNKTktqakZ1d1c3bmdJTms5YmFsVU9sUkJWSHJEMVhR#security-filters// Keyboard shortcuts that are active when the focus is in the editor
{ "key": "home",            "command": "cursorHome",                  "when": "editorTextFocus" },
{ "key": "shift+home",      "command": "cursorHomeSelect",            "when": "editorTextFocus" },

// Keyboard shortcuts that are complementary
{ "key": "f5",              "command": "workbench.action.debug.continue", "when": "inDebugMode" },
{ "key": "f5",              "command": "workbench.action.debug.start",    "when": "!inDebugMode" },

// Global keyboard shortcuts
{ "key": "ctrl+f",          "command": "actions.find" },
{ "key": "alt+left",        "command": "workbench.action.navigateBack" },
{ "key": "alt+right",       "command": "workbench.action.navigateForward" },

// Global keyboard shortcuts using chords (two separate keypress actions)
{ "key": "ctrl+k enter",    "command": "workbench.action.keepEditor" },
{ "key": "ctrl+k ctrl+w",   "command": "workbench.action.closeAllEditors" },
[KeybindingService]: / Received  keydown event - modifiers: [meta], code: MetaLeft, keyCode: 91, key: Meta
[KeybindingService]: | Converted keydown event - modifiers: [meta], code: MetaLeft, keyCode: 57 ('Meta')
[KeybindingService]: \ Keyboard event cannot be dispatched.
[KeybindingService]: / Received  keydown event - modifiers: [meta], code: Slash, keyCode: 191, key: /
[KeybindingService]: | Converted keydown event - modifiers: [meta], code: Slash, keyCode: 85 ('/')
[KeybindingService]: | Resolving meta+[Slash]
[KeybindingService]: \ From 2 keybinding entries, matched editor.action.commentLine, when: editorTextFocus && !editorReadonly, source: built-in.
consentInformation = UserMessagingPlatform.getConsentInformation(context);content://media/external/downloads/1000015133---
título: Comandos de API · Documentación SSL/TLS de Cloudflare
Descripción: Utilice los siguientes comandos de la API para administrar certificados avanzados. Si es la primera vez que utiliza nuestra API, consulte la documentación de la misma.
Última actualización: 2025-04-22T13:13:58.000Z
URL de origen:
  html: https://developers.cloudflare.com/ssl/edge-certificates/advanced-certificate-manager/api-commands/
  md: https://developers.cloudflare.com/ssl/edge-certificates/advanced-certificate-manager/api-commands/index.md
---

Utilice los siguientes comandos de API para administrar certificados avanzados. Si es la primera vez que utiliza nuestra API, consulte nuestra documentación de API.

| Comando | Método | Punto final | Notas adicionales |
| - | - | - | - |
| [Solicitar certificado avanzado](https://developers.cloudflare.com/api/resources/ssl/subresources/certificate_packs/methods/create/) | `POST` | `zones/<<ZONE_ID>>/ssl/certificate_packs/order` | |
| [Reiniciar la validación del certificado](https://developers.cloudflare.com/api/resources/ssl/subresources/certificate_packs/methods/edit/) | `PATCH` | `zones/<<ZONE_ID>>/ssl/certificate_packs/<<ID>>` | Para un paquete de certificados en estado `validation_timed_out`. |
| [Eliminar paquete de certificados](https://developers.cloudflare.com/api/resources/ssl/subresources/certificate_packs/methods/delete/) | `ELIMINAR` | `zones/<<ZONA_ID>>/ssl/certificate_packs/<<ID>>` | |
| [Enumerar paquetes de certificados en una zona](https://developers.cloudflare.com/api/resources/ssl/subresources/certificate_packs/methods/list/) | `GET` | `zones/<<ZONE_ID>>/ssl/certificate_packs?status=all` | Esta llamada API devuelve todos los paquetes de certificados para un dominio (universal, personalizado y avanzado). |
| Lista de configuraciones de Cipher Suite: [Obtener configuración de zona](https://developers.cloudflare.com/api/resources/zones/subresources/settings/methods/get/) con `ciphers` como nombre de configuración en la ruta URI | `GET` | `zones/<<ZONE_ID>>/settings/ciphers` | |
| Cambiar la configuración de Cipher Suite: [Editar configuración de zona](https://developers.cloudflare.com/api/resources/zones/subresources/settings/methods/edit/) con `ciphers` como nombre de configuración en la ruta URI | `PATCH` | `zones/<<ZONE_ID>>/settings/ciphers` | Para restaurar la configuración predeterminada, envíe una matriz en blanco en el parámetro `value`. |Verificación = {
certificado_estado :
Estado actual del certificado.

"inicializando"
"autorizando"
"activo"
"venció"
"emisor"
"tiempo_agotado"
"pendiente de implementación"
brand_check : booleano
Opcional
La autoridad certificadora está revisando el pedido manualmente.

cert_pack_uuid : cadena
Opcional
UUID del paquete de certificados.

Firma : 
Opcional
Algoritmo de firma del certificado.

"ECDSAWithSHA256"
"SHA1 con RSA"
"SHA256 con RSA"
método_de_validación :Método de validación
Opcional
Método de validación en uso para un pedido de paquete de certificados.

información_de_verificación : { nombre_del_registro , destino_del_registro }
Opcional
Información de verificación requerida del certificado.

estado_de_verificación : booleano
Opcional
Estado de la información de verificación requerida, se omite si se desconoce el estado de verificación.

tipo_de_verificación : 
Opcional
Método de verificación.

"cname"
"metaetiqueta"
}---
title: API commands · Cloudflare SSL/TLS docs
description: Use the following API commands to manage advanced certificates. If you are using our API for the first time, review our API documentation.
lastUpdated: 2025-04-22T13:13:58.000Z
source_url:
  html: https://developers.cloudflare.com/ssl/edge-certificates/advanced-certificate-manager/api-commands/
  md: https://developers.cloudflare.com/ssl/edge-certificates/advanced-certificate-manager/api-commands/index.md
---

Use the following API commands to manage advanced certificates. If you are using our API for the first time, review our [API documentation](https://developers.cloudflare.com/fundamentals/api/).

| Command | Method | Endpoint | Additional notes |
| - | - | - | - |
| [Order advanced certificate](https://developers.cloudflare.com/api/resources/ssl/subresources/certificate_packs/methods/create/) | `POST` | `zones/<<ZONE_ID>>/ssl/certificate_packs/order` | |
| [Restart certificate validation](https://developers.cloudflare.com/api/resources/ssl/subresources/certificate_packs/methods/edit/) | `PATCH` | `zones/<<ZONE_ID>>/ssl/certificate_packs/<<ID>>` | For a Certificate Pack in a `validation_timed_out` status. |
| [Delete certificate pack](https://developers.cloudflare.com/api/resources/ssl/subresources/certificate_packs/methods/delete/) | `DELETE` | `zones/<<ZONE_ID>>/ssl/certificate_packs/<<ID>>` | |
| [List certificate packs in a zone](https://developers.cloudflare.com/api/resources/ssl/subresources/certificate_packs/methods/list/) | `GET` | `zones/<<ZONE_ID>>/ssl/certificate_packs?status=all` | This API call returns all certificate packs for a domain (Universal, Custom, and Advanced). |
| List Cipher Suite settings: [Get zone setting](https://developers.cloudflare.com/api/resources/zones/subresources/settings/methods/get/) with `ciphers` as the setting name in the URI path | `GET` | `zones/<<ZONE_ID>>/settings/ciphers` | |
| Change Cipher Suite settings: [Edit zone setting](https://developers.cloudflare.com/api/resources/zones/subresources/settings/methods/edit/) with `ciphers` as the setting name in the URI path | `PATCH` | `zones/<<ZONE_ID>>/settings/ciphers` | To restore default settings, send a blank array in the `value` parameter. |
{
  "errors": [
    {
      "code": 1000,
      "message": "message",
      "documentation_url": "documentation_url",
      "source": {
        "pointer": "pointer"
      }
    }
  ],
  "messages": [
    {
      "code": 1000,
      "message": "message",
      "documentation_url": "documentation_url",
      "source": {
        "pointer": "pointer"
      }
    }
  ],
  "success": true,
  "result": {
    "id": "023e105f4ecef8ad9ca31a8372d0c353",
    "certificate_authority": "lets_encrypt",
    "cloudflare_branding": false,
    "hosts": [
      "example.com",
      "*.example.com",
      "www.example.com"
    ],
    "status": "initializing",
    "type": "advanced",
    "validation_method": "txt",
    "validity_days": 14
  }
}Sobre <ZeroRTT|DDoS avanzado| { id , modificado_el , valor } | 57 más... >
conseguir
/ zonas / {zone_id} / configuraciones / {setting_id}{
  "errors": [
    {
      "code": 1000,
      "message": "message",
      "documentation_url": "documentation_url",
      "source": {
        "pointer": "pointer"
      }
    }
  ],
  "messages": [
    {
      "code": 1000,
      "message": "message",
      "documentation_url": "documentation_url",
      "source": {
        "pointer": "pointer"
      }
    }
  ],
  "success": true,
  "result": {
    "id": "0rtt",
    "value": "on"
  }
}---
title: Overview · Cloudflare DMARC Management docs
description: Stop brand impersonation.
lastUpdated: 2025-03-14T16:33:10.000Z
source_url:
  html: https://developers.cloudflare.com/dmarc-management/
  md: https://developers.cloudflare.com/dmarc-management/index.md
---

Stop brand impersonation.

Available on all plans

Cloudflare DMARC Management (beta) helps you track every source that is sending emails from your domain and review [Domain-based Message Authentication Reporting and Conformance (DMARC)](https://www.cloudflare.com/learning/dns/dns-records/dns-dmarc-record/) reports for each source. DMARC reports will help you understand if messages sent from your domain are passing DMARC authentication, [DomainKeys Identified Mail (DKIM)](https://www.cloudflare.com/learning/dns/dns-records/dns-dkim-record/) authentication, and [Sender Policy Framework (SPF)](https://www.cloudflare.com/learning/dns/dns-records/dns-spf-record/) policies.

DMARC Management (beta) is available to all Cloudflare customers with [Cloudflare DNS](https://developers.cloudflare.com/dns/).

***

## Related products

**[Area 1 Email Security](https://developers.cloudflare.com/email-security/)**

Stop phishing attacks with Area 1 cloud-native email security service.

**[Cloudflare DNS](https://developers.cloudflare.com/dns/)**

Fast, resilient and easy-to-manage DNS service.
import hmac
import base64
import time
import urllib.parse
from hashlib import sha256

message = "/images/cat.jpg"
secret = "mysecrettoken"
separator = "verify"
timestamp = str(int(time.time()))
digest = hmac.new((secret).encode('utf8'), "{}{}".format(message, timestamp).encode('utf8'), sha256)
token = urllib.parse.quote_plus(base64.b64encode(digest.digest()))
print("{}={}-{}".format(separator, timestamp, token))---
title: Public buckets · Cloudflare R2 docs
description: Public Bucket is a feature that allows users to expose the contents of their R2 buckets directly to the Internet. By default, buckets are never publicly accessible and will always require explicit user permission to enable.
lastUpdated: 2025-05-06T15:18:12.000Z
source_url:
  html: https://developers.cloudflare.com/r2/buckets/public-buckets/
  md: https://developers.cloudflare.com/r2/buckets/public-buckets/index.md
---

Public Bucket is a feature that allows users to expose the contents of their R2 buckets directly to the Internet. By default, buckets are never publicly accessible and will always require explicit user permission to enable.

Public buckets can be set up in either one of two ways:

* Expose your bucket as a custom domain under your control.
* Expose your bucket using a Cloudflare-managed `https://r2.dev` subdomain for non-production use cases.

These options can be used independently. Enabling custom domains does not require enabling `r2.dev` access.

To use features like WAF custom rules, caching, access controls, or bot management, you must configure your bucket behind a custom domain. These capabilities are not available when using the `r2.dev` development url.

Note

Currently, public buckets do not let you list the bucket contents at the root of your (sub) domain.

## Custom domains

### Caching

Domain access through a custom domain allows you to use [Cloudflare Cache](https://developers.cloudflare.com/cache/) to accelerate access to your R2 bucket.

Configure your cache to use [Smart Tiered Cache](https://developers.cloudflare.com/cache/how-to/tiered-cache/#smart-tiered-cache) to have a single upper tier data center next to your R2 bucket.

Note

By default, only certain file types are cached. To cache all files in your bucket, you must set a Cache Everything page rule.

For more information on default Cache behavior and how to customize it, refer to [Default Cache Behavior](https://developers.cloudflare.com/cache/concepts/default-cache-behavior/#default-cached-file-extensions)

### Access control

To restrict access to your custom domain's bucket, use Cloudflare's existing security products.

* [Cloudflare Zero Trust Access](https://developers.cloudflare.com/cloudflare-one/applications/configure-apps): Protects buckets that should only be accessible by your teammates.
* [Cloudflare WAF Token Authentication](https://developers.cloudflare.com/waf/custom-rules/use-cases/configure-token-authentication/): Restricts access to documents, files, and media to selected users by providing them with an access token.

Warning

Disable public access to your [`r2.dev` subdomain](#disable-public-development-url) when using products like WAF or Cloudflare Access. If you do not disable public access, your bucket will remain publicly available through your `r2.dev` subdomain.

### Minimum TLS Version

To specify the minimum TLS version of a custom hostname of an R2 bucket, you can issue an API call to edit [R2 custom domain settings](https://developers.cloudflare.com/api/resources/r2/subresources/buckets/subresources/domains/subresources/custom/methods/update/).

## Add your domain to Cloudflare

The domain being used must have been added as a [zone](https://developers.cloudflare.com/fundamentals/setup/accounts-and-zones/#zones) in the same account as the R2 bucket.

* If your domain is already managed by Cloudflare, you can proceed to [Connect a bucket to a custom domain](https://developers.cloudflare.com/r2/buckets/public-buckets/#connect-a-bucket-to-a-custom-domain).
* If your domain is not managed by Cloudflare, you need to set it up using a [partial (CNAME) setup](https://developers.cloudflare.com/dns/zone-setups/partial-setup/) to add it to your account.

Once the domain exists in your Cloudflare account (regardless of setup type), you can link it to your bucket.

## Connect a bucket to a custom domain

1. Go to **R2** and select your bucket.
2. On the bucket page, select **Settings**.
3. Under **Custom Domains**, select **Add**.
4. Enter the domain name you want to connect to and select **Continue**.
5. Review the new record that will be added to the DNS table and select **Connect Domain**.

Your domain is now connected. The status takes a few minutes to change from **Initializing** to **Active**, and you may need to refresh to review the status update. If the status has not changed, select the *...* next to your bucket and select **Retry connection**.

To view the added DNS record, select **...** next to the connected domain and select **Manage DNS**.

Note

If the zone is on an Enterprise plan, make sure that you [release the zone hold](https://developers.cloudflare.com/fundamentals/setup/account/account-security/zone-holds/#release-zone-holds) before adding the custom domain.

A zone hold would prevent the custom subdomain from activating.

## Disable domain access

Disabling a domain will turn off public access to your bucket through that domain. Access through other domains or the managed `r2.dev` subdomain are unaffected. The specified domain will also remain connected to R2 until you remove it or delete the bucket.

To disable a domain:

1. In **R2**, select the bucket you want to modify.
2. On the bucket page, Select **Settings**, go to **Custom Domains**.
3. Next to the domain you want to disable, select **...** and **Disable domain**.
4. The badge under **Access to Bucket** will update to **Not allowed**.

## Remove domain

Removing a custom domain will disconnect it from your bucket and delete its configuration from the dashboard. Your bucket will remain publicly accessible through any other enabled access method, but the domain will no longer appear in the connected domains list.

To remove a domain:

1. In **R2**, select the bucket you want to modify.
2. On the bucket page, Select **Settings**, go to **Custom Domains**.
3. Next to the domain you want to disable, select **...** and **Remove domain**.
4. Select ‘Remove domain’ in the confirmation window. This step also removes the CNAME record pointing to the domain. You can always add the domain again.

## Public development URL

Expose the contents of this R2 bucket to the internet through a Cloudflare-managed r2.dev subdomain. This endpoint is intended for non-production traffic.

Note

Public access through `r2.dev` subdomains are rate limited and should only be used for development purposes.

To enable access management, Cache and bot management features, you must set up a custom domain when enabling public access to your bucket.

Avoid creating a CNAME record pointing to the `r2.dev` subdomain. This is an **unsupported access path**, and we cannot guarantee consistent reliability or performance. For production use, [add your domain to Cloudflare](#add-your-domain-to-cloudflare) instead.

### Enable public development url

When you enable public development URL access for your bucket, its contents become available on the internet through a Cloudflare-managed `r2.dev` subdomain.

To enable access through `r2.dev` for your buckets:

1. In **R2**, select the bucket you want to modify.
2. On the bucket page, select **Settings**.
3. Under **Public Development URL**, select **Enable**.
4. In **Allow Public Access?**, confirm your choice by typing ‘allow’ to confirm and select **Allow**.
5. You can now access the bucket and its objects using the Public Bucket URL.

To verify that your bucket is publicly accessible, check that **Public URL Access** shows **Allowed** in you bucket settings.

### Disable public development url

Disabling public development URL access removes your bucket’s exposure through the `r2.dev` subdomain. The bucket and its objects will no longer be accessible via the Public Bucket URL.

If you have connected other domains, the bucket will remain accessible for those domains.

To disable public access for your bucket:

1. In **R2**, select the bucket you want to modify.
2. On the bucket page, select **Settings**.
3. Under **Public Development URL**, select **Disable**.
4. In **Disallow Public Access?**, type ‘disallow’ to confirm and select **Disallow**.
verify=1484063787-9JQB8vP1z0yc5DEBnH6JGWM3mBmvIeMrnnxFi3WtJLE%3D

# excalibur
excalibur is a project that use DarkSword exploit for customize your iPhone, include some feature like Springboard injection, bypass 3 app limit, etc.

## Build IPA on GitHub (no Mac required)

The workflow `.github/workflows/build-ipa.yml` builds on a GitHub-hosted macOS runner. It creates a signed ad-hoc IPA and uploads it as an Actions artifact. It only runs when started manually, so a normal push will not accidentally publish an IPA.

If you will sign the IPA yourself, use **Build unsigned IPA** instead. It needs no Apple Developer account and no GitHub Secrets. Every push to `main` automatically builds `FreeFireDS.ipa` and creates a GitHub Release. Its version is `v<MARKETING_VERSION>.<GitHub run number>` (for example `v1.0.42`), so each build has a unique release. The file cannot be installed until you re-sign it with your own signing tool and certificate.

Before the first run, create these **repository secrets** in GitHub: **Settings → Secrets and variables → Actions → New repository secret**.

| Secret | Value |
| --- | --- |
| `DEVELOPMENT_TEAM` | Your 10-character Apple Developer Team ID. |
| `IOS_CERTIFICATE` | Base64 text of your distribution `.p12` certificate. |
| `IOS_CERTIFICATE_PASSWORD` | Password used when exporting that `.p12`. |
| `IOS_PROVISIONING_PROFILE` | Base64 text of an ad-hoc `.mobileprovision` file for `com.34306.excalibur1`. |
| `IOS_PROVISIONING_PROFILE_NAME` | Exact profile name displayed in Apple Developer. |

You need a paid Apple Developer Program membership to create a distribution certificate and an ad-hoc profile. Register the iPhones/iPads that should install the IPA in the Apple Developer portal, then create the profile for the app ID `com.34306.excalibur1` and those devices.

You can create the certificate request on Windows—no Mac is needed. With OpenSSL installed, generate a key and CSR, upload the CSR when creating an **Apple Distribution** certificate in Apple Developer, download the resulting `.cer`, then make the `.p12` used above:

```powershell
openssl req -new -newkey rsa:2048 -nodes -keyout ios_distribution.key -out ios_distribution.csr -subj "/CN=Your Name/emailAddress=you@example.com"
openssl x509 -inform DER -in ios_distribution.cer -out ios_distribution.pem
openssl pkcs12 -export -out ios_distribution.p12 -inkey ios_distribution.key -in ios_distribution.pem -name "Apple Distribution"
```

On Windows, base64-encode files for a GitHub secret with PowerShell (copy the resulting single line, never commit it):

```powershell
[Convert]::ToBase64String([IO.File]::ReadAllBytes("ios_distribution.p12"))
[Convert]::ToBase64String([IO.File]::ReadAllBytes("Excalibur_AdHoc.mobileprovision"))
```

To run it: open the repository’s **Actions** tab → **Build signed IPA** → **Run workflow**. Download `Excalibur-IPA-<run number>` from the completed run. Choose `publish_release` only if you intentionally want the IPA attached to a public/private GitHub Release.

# TODO list
- Springboard inject tweak
- Sandbox escaped
- 3 app bypass
- Decrypt app to iPA
- Enable JIT(?)
- Overwrite system file (custom overwrite?)
- Custom mobilegestalt (for Dynamic island enabler, hidden feature, etc)
- Memory finder and editor?


# Credit
- wh1te4ever for kfun-darksword project

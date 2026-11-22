# Release signing

How to sign Android release builds with your own key, both on your computer and on GitHub
Actions, without the key ever entering a repository. The commands are for Linux.

## What the key is, and the two rules

Android only installs an update if it's signed with the same key as the installed app. So the
release key **is** the app's identity:

- **Lose it and you can never update your app again.** Users would have to uninstall (losing
  their data) and install a new app.
- **Leak it and someone else can publish "updates" to your app.**

The key never goes in a repository. It lives in one file on your computer, a backup, and your CI's
encrypted secrets.

One key for all your apps is the simple choice. The cost: if it leaks, every app is affected at
once. You can use a separate key per app instead. Either way, you can't change the key of an app
people already have installed.

## 1. Create the key (once)

`keytool` comes with the JDK you use to build (`sudo apt install openjdk-21-jdk-headless` if
it's missing).

Make a private folder for it:

```bash
mkdir -p ~/android-keys && chmod 700 ~/android-keys
```

Make up a long password. This prints a random one. Save it in your password manager **before**
going on:

```bash
openssl rand -base64 24
```

Create the key. It asks for the password (twice), then your name, organisation and so on.
Those answers are written into the certificate, which anyone can read from the APK, so fill in
only what you're happy to make public (a name is enough; Enter skips the rest). Then answer
`yes`.

```bash
keytool -genkeypair -v -keystore ~/android-keys/release.jks -storetype PKCS12 -alias release -keyalg RSA -keysize 4096 -validity 10000
```

That's a 4096-bit RSA key, valid about 27 years, with the alias `release` that the build setup
below expects.

```bash
chmod 600 ~/android-keys/release.jks
```

### Back it up now

Copy `~/android-keys/release.jks` to at least two places that aren't this computer, such as a USB
stick in a drawer and an encrypted cloud backup or a password manager (most can store file
attachments). Keep the password with it. Without both the file and the password, the key is
gone.

## 2. Set up the app's build

### `app/build.gradle.kts`

Above the `android { }` block:

```kotlin
// Release signing. The keystore and its password come from ~/.gradle/gradle.properties
// (releaseStoreFile, releaseStorePassword, optional releaseKeyAlias) or, on CI, from environment
// variables (RELEASE_STORE_FILE, RELEASE_STORE_PASSWORD, RELEASE_KEY_ALIAS). Without them,
// release builds come out unsigned.
fun signingSetting(property: String, variable: String): String? =
    (providers.gradleProperty(property).orNull ?: providers.environmentVariable(variable).orNull)
        ?.takeIf { it.isNotBlank() }

val releaseStoreFile = signingSetting("releaseStoreFile", "RELEASE_STORE_FILE")
    ?.let { if (it.startsWith("~/")) System.getProperty("user.home") + it.substring(1) else it }
```

Inside `android { }`:

```kotlin
    signingConfigs {
        if (releaseStoreFile != null) {
            create("release") {
                storeFile = file(releaseStoreFile)
                storePassword = signingSetting("releaseStorePassword", "RELEASE_STORE_PASSWORD")
                keyAlias = signingSetting("releaseKeyAlias", "RELEASE_KEY_ALIAS") ?: "release"
                // PKCS12 keystores (keytool's default) use one password for the store and the key
                keyPassword = storePassword
            }
        }
    }

    buildTypes {
        release {
            signingConfig = signingConfigs.findByName("release")
            // ...
        }
    }
```

| Gradle property (`~/.gradle/gradle.properties`) | or environment variable (CI) | |
|---|---|---|
| `releaseStoreFile` | `RELEASE_STORE_FILE` | path to the keystore (`~/` allowed) |
| `releaseStorePassword` | `RELEASE_STORE_PASSWORD` | its password |
| `releaseKeyAlias` | `RELEASE_KEY_ALIAS` | optional, default `release` |

Anyone who builds the app without these settings still gets a working build, just an unsigned
release APK.

Also add `*.jks` and `*.keystore` to `.gitignore`, so a key can't be committed by accident.

### Release builds with the open SDK

With `isMinifyEnabled = true`, R8 stops with "Missing class android.…" errors when building
against this repository's open SDK: its `android.jar` still names some hidden framework classes.
They're part of Android itself, present on every device, so add this to `app/proguard-rules.pro`
(and list that file in `proguardFiles(...)`):

```
-dontwarn android.**
```

Google's SDK doesn't need this.

## 3. Sign on your computer

Tell Gradle where the key is, once for all projects, in `~/.gradle/gradle.properties` (create the
file if it doesn't exist). Put your real password in place of `PASSWORD`:

```properties
releaseStoreFile=~/android-keys/release.jks
releaseStorePassword=PASSWORD
```

The password is stored in plain text there, so only you should be able to read it:

```bash
chmod 600 ~/.gradle/gradle.properties
```

Then:

```bash
./gradlew assembleRelease
```

The signed APK is `app/build/outputs/apk/release/app-release.apk`. Without the two settings you
get `app-release-unsigned.apk`, which phones refuse to install.

Check which key signed an APK:

```bash
keytool -printcert -jarfile app/build/outputs/apk/release/app-release.apk
```

The `SHA256:` line is your key's fingerprint. Write it down: it's how you, app stores or users can
tell a genuine APK from a fake one.

## 4. Sign on GitHub Actions

### Secrets

Each repository needs two secrets (personal accounts can't share secrets between repositories;
organizations can):

| Secret | Value |
|---|---|
| `RELEASE_KEYSTORE` | the keystore file, base64-encoded |
| `RELEASE_STORE_PASSWORD` | the password |

With the GitHub CLI (`sudo apt install gh`, then `gh auth login` once), replacing `OWNER/REPO`:

```bash
base64 -w0 ~/android-keys/release.jks | gh secret set RELEASE_KEYSTORE --repo OWNER/REPO
```

```bash
gh secret set RELEASE_STORE_PASSWORD --repo OWNER/REPO
```

The second command asks for the password, so it doesn't end up in your shell history.

Or on the website: the repository's **Settings → Secrets and variables → Actions → New
repository secret**, once per secret. For `RELEASE_KEYSTORE`, paste the output of
`base64 -w0 ~/android-keys/release.jks` (one long line).

### Workflow

In `.github/workflows/build.yml`, before the Gradle step:

```yaml
      # Release key, if the repository has the secrets. Without them the release APK comes out
      # unsigned.
      - name: Release signing key
        env:
          RELEASE_KEYSTORE: ${{ secrets.RELEASE_KEYSTORE }}
        if: env.RELEASE_KEYSTORE != ''
        run: |
          echo "$RELEASE_KEYSTORE" | base64 -d > "$RUNNER_TEMP/release.jks"
          echo "RELEASE_STORE_FILE=$RUNNER_TEMP/release.jks" >> "$GITHUB_ENV"
```

On the Gradle step, add `assembleRelease` and the password:

```yaml
      - name: Build
        env:
          RELEASE_STORE_PASSWORD: ${{ secrets.RELEASE_STORE_PASSWORD }}
        run: ./gradlew --no-daemon assembleRelease
```

After it:

```yaml
      - name: Remove the release key
        if: always()
        run: rm -f "$RUNNER_TEMP/release.jks"
```

Then upload `app/build/outputs/apk/release/app-release.apk` with `actions/upload-artifact`, or
attach it to a release.

Secrets are never given to pull requests from forks, so those still build, just unsigned.

## 5. Builds made by AI coding agents

Don't give the key to a cloud coding agent such as Claude Code on the web. A signing key is a
file the build has to read inside the agent's sandbox, and anything stored there can be read by
every command the agent runs, including code from repositories and the internet. (Such
environments may offer "secrets" for API tokens that are added to network requests on the way
out, so the agent never sees them. That works for API keys, not for a file a build has to read.)

You don't need to: when the agent pushes a branch, GitHub Actions builds and signs it. Download
the signed APK from that run's artifacts.

Anyone (or any agent) with push access to the repository could change a workflow on a branch to
print the key. GitHub hides secrets in logs, but not once they're re-encoded. For more
protection, put the two secrets in a GitHub *environment* limited to the `main` branch and
release tags (**Settings → Environments**), and give the build job `environment:` with its name.
Then only those builds are signed.

## 6. Switching an installed app to release builds

An app installed from debug builds is signed with a debug key, and Android won't install a
release-signed APK over it ("App not installed" / "package conflicts"). Uninstall it once, then
install the release build. Uninstalling deletes the app's data, so note its settings first.

After that, each release build installs over the previous one. Debug builds still use the debug
key, so on a given phone stick to either debug or release builds of an app.

## 7. F-Droid

F-Droid builds apps from source and normally signs them with **its own** key. Your key then only
matters for APKs you distribute yourself (GitHub releases, Obtainium), and the two versions can't
update each other. To have F-Droid ship your signature instead, ask for reproducible builds when
submitting: F-Droid builds the app itself, copies the signature from your APK onto its unsigned
build, and publishes your signed APK only if the result verifies, which needs the two builds to
be identical apart from the signature.

What that takes:

- **Build with what F-Droid builds with.** F-Droid's build server compiles against Google's SDK
  platform and build-tools, which it installs with its own
  [sdkmanager](https://gitlab.com/fdroid/sdkmanager) from Google's servers, with Debian's JDK
  (21 by default). The open SDK from this repository does **not** give identical APKs: its
  `android.jar` has thousands of hidden classes and keeps hidden superclasses, so D8 and R8 output
  differ. Install the same packages with F-Droid's tool, for example in a GitHub Actions workflow:

  ```yaml
      - name: Install the SDK with F-Droid's sdkmanager
        run: |
          sudo rm -rf /usr/local/lib/android
          pipx install sdkmanager==0.7.2
          sdkmanager "platforms;android-35" "build-tools;34.0.0"
  ```

  with `ANDROID_HOME` set for the job, and the platform and build-tools versions your app's
  `compileSdk` and Android Gradle Plugin use.
- **No dependency metadata blob**, which is encrypted for Google, in `android { }`:

  ```kotlin
      dependenciesInfo {
          includeInApk = false
          includeInBundle = false
      }
  ```

- **Check it yourself** with F-Droid's [apksigcopier](https://github.com/obfusk/apksigcopier),
  against an unsigned build of the same commit (`./gradlew assembleRelease` without the key
  settings):

  ```bash
  apksigcopier compare app-release.apk --unsigned app-release-unsigned.apk
  ```

  No output and exit status 0 means F-Droid's check would pass. APKs signed by the Android Gradle
  Plugin, as set up above, pass it. F-Droid's docs warn about `apksigner` from build-tools 35 or
  later, which only matters if you sign with that tool instead.

F-Droid's [Reproducible Builds](https://f-droid.org/docs/Reproducible_Builds/) page lists other
things that can make builds differ (baseline profiles, build paths, native code).

### Readable crash traces

R8 renames classes and methods in release builds, so crash traces read `a.b.c()`. For an open
source app that hides nothing, so you can turn renaming off and keep line numbers, in
`app/proguard-rules.pro`; shrinking and optimization stay on, and builds stay reproducible:

```
-dontobfuscate
-keepattributes SourceFile,LineNumberTable
```

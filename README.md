# Blazium Game Build Action

Prerequisites:
- `export_presets.cfg`: A file created with what targets you want to export:
  * For Android, enable AAB.
  * For Windows, select what architecture you want for each export.
  * For Linux, select what architecture you want for each export.

Reusable Action that build a Blazium game. Example usage:

```yml
jobs:
  build:
    runs-on: ${{ matrix.platform.os }}
    strategy:
      fail-fast: false
      matrix:
        platform: [
          "Windows Desktop x86_64",
          "Windows Desktop x86_32",
          "Windows Desktop arm64",
          "Windows Desktop arm32",
          "Linux x86_64",
          "Linux x86_32",
          "macOS",
          "iOS",
          "Android",
          "Web"
        ]

    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Increment Version
        run: |
          python deploy/update_version.py

      - name: Build Game
        uses: blazium-engine/build-game@master
        with:
          game-name: ${{ env.GAME_NAME }}
          platform-name: ${{ matrix.platform }}
          secret-macos-build-certificate-base64: ${{ secrets.BUILD_CERTIFICATE_BASE64 }}
          secret-p12-password: ${{ secrets.P12_PASSWORD }}
          secret-keychain-password: ${{ secrets.KEYCHAIN_PASSWORD }}
          secret-ios-distribution-certificate-base64: ${{ secrets.DISTRIBUTION_CERTIFICATE_BASE64 }}
          secret-ios-deploy-provision-profile-ios-base64: ${{ secrets.DEPLOY_PROVISION_PROFILE_IOS_BASE64 }}
          secret-apple-id: ${{ secrets.APPLE_ID }}
          secret-apple-team-id: ${{ secrets.APPLE_TEAM_ID }}
          secret-apple-password: ${{ secrets.APP_SPECIFIC_PASSWORD }}
```

# Apple Secrets configuration

The deploy secrets are needed for deploying to App Store or Mac App Store.

# Apple Mac Flow for Export

## 1. How to get Certificate P12 to Mac

Prerequisites:

- P12 Password (You create this manually, keep it safe)

First go to `Certificates, Identifiers & Profiles` -> `Certificates`:

- `https://developer.apple.com/account/resources/certificates/list`

Then, download the certificate which was previously generated using a file called `CertificateSigningRequest.certSigningReque` (PRIVATE KEY). The private key is created by following this link `https://developer.apple.com/help/account/create-certificates/create-a-certificate-signing-request`. The public key is unusable without the private key for our use cases.

After this, import the public key and also import the privat key. Now, go to XCode -> Settings -> Accounts -(Click the team to which the certificate belongs to)> Manage Certificates... -> (Right Click the Certificate) it has to match by name, DO NOT import multiple certificates with same name -> Export Certificate - put password> Obtain a P12 file

Certificate Types:
- Developer ID Application - For Mac Binary Exports
- Mac App Distribution - For Mac Binary to be sent to Mac App Store (inside the installer)
- Mac Installer Distribution - For Mac Installer (contains the Mac Binary) to be sent to the Mac App Store

### Generate base64 from Certificate P12

To use them in CI/CD you need to make them in base64 so they can be put in env vars:
```sh
# eg.
base64 -i DeveloperID.p12 > certificate_development_id_base64.txt
```

## 2. Obtain the certificates

Prerquisites:
- Developer ID Certificate
- P12 Password
- Keychain Password (you set this)

### How to list certificates installed

You installed the certificate by a file, but now you need a name to use it. In order to get that name, you can do:

```sh
security find-identity -v -p codesigning
```

## 3. Build the mac app without signing and without notarization

There are 3 types of mac apps:
- .app: Folder with files
- .dmg: Virtual Volume, basically a single file
- .pkg: Installer

You want to use the .app and create a .pkg with it. Using .dmg is also possible but more difficult.

## 4. Sign the app

Prerequisite:

- APPLE_ID: Your apple id (usually email)
- APP_SPECIFIC_PASSWORD: https://support.apple.com/en-us/102654
- APPLE_TEAM_ID: Your apple team id (usually a number, eg. 12345678)
- entitlements file:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
	<dict>
    <key>com.apple.security.app-sandbox</key>
    <true/>
    <key>com.apple.security.files.user-selected.read-write</key>
    <true/>
    <key>com.apple.security.network.client</key>
    <true/>
  </dict>
</plist>
```

You need to sign the app using the `Developer ID Application` for your team if you want to export it (outside of Mac App Store).

# Apple Mac Flow for Publish

## 1. Provisioning Profile

In order to deploy to Mac App Store you need a `Mac App Store Connect` Distribution Profile from `https://developer.apple.com/account/resources/profiles/add`. The distribution profile is a text file that contains also some entitlements, these will be added to the entitlements file later on.

How to get it:
- Go to Certificates, Identifiers & Profiles: `https://developer.apple.com/account/resources/profiles/add`. Add a provisioning profile of type `Mac App Store Connect` at the `Distribution` section. Select App Id that matches your game. This is how it will know to link the provisioning profile with the certificate. Then click Next, Next..

## 2. Obtain the certificates

Prerquisites:
- Mac App Distribution
- Mac Installer Distribution
- P12 Password
- Keychain Password (you set this)
- Provisioning Profile of type `Mac App Store Connect`

## 3. Code Sign

Prerequisites:
- `DEPLOY_SIGNING_IDENTITY` you obtained from Step 2.
- `entitlements-deploy.plist`. Do not forget to add the `app-sandbox` entitlement, you will see it below added. You also need to embed in the entitlements the provisioning profile entitlements. They will be located in the provisioning profile entitlements section, eg.:
```xml
<key>Entitlements</key>
<dict>
      
  <key>com.apple.application-identifier</key>
  <string>9W3XCW26P5.app.blazium.editor</string>
      
  <key>keychain-access-groups</key>
  <array>
  <string>9W3XCW26P5.*</string>
  </array>
      
  <key>com.apple.developer.team-identifier</key>
  <string>9W3XCW26P5</string>
  
</dict>
```
From here, you only want to copy the inside of the `dict`. You copy and put it in the entitlements file. Eg, if this was the initial entitlements file:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
	<dict>
    <key>com.apple.security.app-sandbox</key>
    <true/>
    <key>com.apple.security.files.user-selected.read-write</key>
    <true/>
    <key>com.apple.security.network.client</key>
    <true/>
  </dict>
</plist>
```

You would copy and would have:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
	<dict>
    <!-- Copy start -->
    <key>com.apple.application-identifier</key>
		<string>9W3XCW26P5.app.blazium.hangman</string>
    <key>keychain-access-groups</key>
		<array>
    <string>9W3XCW26P5.*</string>
		</array>
    <key>com.apple.developer.team-identifier</key>
		<string>9W3XCW26P5</string>
    <!-- Copy end -->
    <key>com.apple.security.app-sandbox</key>
    <true/>
    <key>com.apple.security.files.user-selected.read-write</key>
    <true/>
    <key>com.apple.security.network.client</key>
    <true/>
  </dict>
</plist>
```

## 4. Make an installer

Prerequisites:
- `INSTALL_SIGNING_IDENTITY` you obtained from Step 2.

## 5. Deploy to Test Flight

Prerequisites:
- `APPLE_ID`: Your apple id (usually email)
- `APP_SPECIFIC_PASSWORD`: https://support.apple.com/en-us/102654


# Apple iOS Flow for Publish

This flow works pretty well from Godot. All you need to do is set in Godot same stuff in export window as for actions:

## 1. Obtain certificates and provisioning profile

Prerequisites:
- `DISTRIBUTION_CERTIFICATE_BASE64`: Apple Distribution Certificate
- `P12_PASSWORD`: Password used to encrypt P12 Certificates
- `DEPLOY_PROVISION_PROFILE_IOS_BASE64`: iOS Deploy Certificate for App
- `KEYCHAIN_PASSWORD`: Keychain Password (you enter this manually to what you want)

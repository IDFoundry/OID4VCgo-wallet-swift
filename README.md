# OID4VCgo-wallet-swift

`OID4VCWallet`: the Swift package of [OID4VCgo](https://github.com/IDFoundry/OID4VCgo)'s
mobile wallet, for iOS 15+ and macOS 13+. It receives credentials over
OpenID4VCI 1.0 and presents them over OpenID4VP 1.0, under HAIP 1.0, with
SD-JWT VC and ISO mdoc credentials. OID4VCgo is OpenID Certified for
those roles. It also presents an mdoc to Safari's Digital Credentials API
(`org-iso-mdoc`, from an iOS document provider extension), and in person
over Bluetooth (ISO/IEC 18013-5), as the holder or as the reader,
engaged by a QR code. NFC engagement is in the Kotlin library only:
an iOS wallet can't answer an NFC tap here.

The protocols run in OID4VCgo's Go code, compiled with gomobile into the
`Mobile` XCFramework this package links. Your app owns the keys, the
storage and the UI:
- **Keys:** `KeychainKeyStore` keeps them in the Secure Enclave. Holder
  keys can require Face ID for each use.
- **Storage:** `FileCredentialStore` uses complete file protection and
  is excluded from backup. Its writes lock a file in its directory, so
  an app and its extension can share one in an App Group.
- **Wallet Provider:** your backend attests the wallet, for issuers
  that ask for attestations, as HAIP issuers do. `HttpWalletProvider`
  is the client of one built on OID4VCgo's `walletprovider`, with App
  Attest evidence; for another backend you implement `WalletProvider`,
  making the checks the Getting started article lists. Pass `nil` for
  issuers that don't ask. Issuance follows each issuer's metadata, or
  with `issuanceProfile: .haip` refuses issuers outside HAIP.

## Install

```swift
.package(url: "https://github.com/IDFoundry/OID4VCgo-wallet-swift", from: "0.37.0")
```

Then, in your app's Info.plist and capabilities:
- `NSFaceIDUsageDescription`, as holder keys ask for Face ID at each
  use; without it, iOS asks for the passcode instead.
- A URL type (`CFBundleURLTypes`) for your redirect URI's private-use
  scheme (`com.example.wallet`), registered with issuers, and for the
  `openid-credential-offer` and `openid4vp` schemes your app opens.
- The App Attest capability, for `HttpWalletProvider`.
- `NSBluetoothAlwaysUsageDescription`, for in-person presentation.

## Use

```swift
import Foundation
import OID4VCWallet

// Your Wallet Provider, built on OID4VCgo's walletprovider, which attests
// the wallet and its keys after checking App Attest evidence. A backend
// of your own needs a WalletProvider of your own: the Getting started
// article lists the checks it makes.
func makeWallet(issuerRoots: String, verifierRoots: String, crls: String) throws -> (Wallet, KeychainKeyStore, HttpWalletProvider) {
    let keys = KeychainKeyStore()                  // Secure Enclave keys
    let provider = try HttpWalletProvider(baseURL: URL(string: "https://provider.example.com/wp")!,
                                          instances: FileInstanceRegistry.standard())
    let wallet = try Wallet(
        configuration: WalletConfiguration(
            clientID: "my-wallet",
            redirectURI: "com.example.wallet:/callback",  // a URL scheme in your Info.plist
            issuerRoots: issuerRoots,                      // PEM: the issuers you trust
            verifierRoots: verifierRoots,                  // PEM: the Verifiers you answer
            crls: crls,
            revocation: [.issuer, .status, .verifier]),
        keyStore: keys,
        credentialStore: try FileCredentialStore.standard(),
        provider: provider)
    return (wallet, keys, provider)
}

// signIn opens the issuer's page in an ASWebAuthenticationSession and
// returns the URL it redirects back to.
func example(wallet: Wallet, offerLink: String, requestLink: String, pin: String,
             signIn: (URL) async throws -> URL) async throws {
    // Receive: an openid-credential-offer:// link.
    let issuance = try await wallet.startIssuance(offer: offerLink)
    if issuance.offer.grant == .preAuthorizedCode {
        try await issuance.redeemPreAuthorizedCode(pin: issuance.offer.txCode == nil ? "" : pin)
    } else {
        let url = try await issuance.beginAuthorization()
        let redirect = try await signIn(url)
        try await issuance.completeAuthorization(redirect: redirect)
    }
    let received = try await issuance.requestCredentials()
    try await issuance.close()
    _ = received

    // Present: an openid4vp:// link. presentation.queries lists what each
    // of the Verifier's queries can be answered with, for the holder to
    // choose; defaultSelection() is the first choice for each.
    let presentation = try await wallet.startPresentation(request: requestLink)
    let selection = try await presentation.defaultSelection()
    let disclosed = try await presentation.preview(selection: selection)   // show the holder
    _ = disclosed
    _ = try await presentation.respond(selection: selection)              // Face ID here
}

// At launch, before any issuance: delete keys an interrupted issuance or
// registration left behind.
func atLaunch(wallet: Wallet, keys: KeychainKeyStore, provider: HttpWalletProvider) async throws {
    try await wallet.sweepOrphanedKeys(in: keys)
    try await provider.sweepOrphanedRegistrationKeys()
}

func handle(_ error: Error) -> String {
    guard let e = error as? WalletError else { return error.localizedDescription }
    return e.isRetryable ? e.localizedDescription + " Try again." : e.localizedDescription
}
```

Errors are `WalletError`s. Each has:
- a stable `code`
- the remote party's own OAuth code in `protocolError`, for example
  `invalid_grant` for a wrong PIN
- `isRetryable`
- a `localizedDescription` you can show the holder

## Documentation

- [The OID4VCWallet documentation](https://idfoundry.github.io/OID4VCgo-wallet-swift/0.37.0/documentation/oid4vcwallet/): getting started, an article
  for each task, and the API reference.
- [The integration checklist](https://idfoundry.github.io/OID4VCgo-wallet-swift/0.37.0/documentation/oid4vcwallet/integrationchecklist): every step
  and rule in order, each with how to check it.
- What's new: each [release](https://github.com/IDFoundry/OID4VCgo-wallet-swift/releases)'s notes.
- [OID4VCgo's `mobile/`](https://github.com/IDFoundry/OID4VCgo/tree/main/mobile):
  what each platform supports, in-person presentation, the Go side and
  its [ABI](https://github.com/IDFoundry/OID4VCgo/blob/main/mobile/ABI.md).
- [`mobile/MOBILE.md`](https://github.com/IDFoundry/OID4VCgo/blob/main/mobile/MOBILE.md):
  the design.
- [The demo wallet app](https://github.com/IDFoundry/OID4VCgo/tree/main/mobile/ios/DemoWallet):
  a complete app on this package, including the document provider.

## For coding assistants

Integrating this package for someone? Follow
[the integration checklist](https://idfoundry.github.io/OID4VCgo-wallet-swift/0.37.0/documentation/oid4vcwallet/integrationchecklist), and read the
documentation's Getting started article, then the article for each
feature, before writing code. Use only the public API those name: never the `Mobile` framework's
generated types directly, and don't reimplement the protocols or the
Bluetooth transport the package carries. In person, this package
engages by QR code only: don't add NFC around it. Match the version of
the documentation you read to the package version you depend on.

## Contributing

This repository is generated: OID4VCgo's `wallet-swift-release`
workflow publishes every release here from
[`mobile/ios`](https://github.com/IDFoundry/OID4VCgo/tree/main/mobile/ios),
README included. Issues and pull requests belong in
[OID4VCgo](https://github.com/IDFoundry/OID4VCgo).

## License

MIT. See [LICENSE](LICENSE).

# OID4VCgo-wallet-swift

`OID4VCWallet`: the Swift package of [OID4VCgo](https://github.com/IDFoundry/OID4VCgo)'s
mobile wallet, for iOS 16+ and macOS 13+. It receives credentials over
OpenID4VCI 1.0 and presents them over OpenID4VP 1.0, under HAIP 1.0, with
SD-JWT VC and ISO mdoc credentials. OID4VCgo is OpenID Certified for
those roles. It also presents an mdoc to Safari's Digital Credentials API
(`org-iso-mdoc`, from an iOS document provider extension), and in person
over Bluetooth (ISO/IEC 18013-5), as the holder or as the reader.

The protocols run in OID4VCgo's Go code, compiled with gomobile into the
`Mobile` XCFramework this package links. Your app owns the keys, the
storage and the UI:
- **Keys:** `KeychainKeyStore` keeps them in the Secure Enclave. Holder
  keys can require Face ID to present.
- **Storage:** `FileCredentialStore` uses complete file protection and
  is excluded from backup.
- **Wallet Provider:** you implement `WalletProvider`, which attests the
  wallet.

## Install

```swift
.package(url: "https://github.com/IDFoundry/OID4VCgo-wallet-swift", from: "0.8.1")
```

In-person presentation needs `NSBluetoothAlwaysUsageDescription` in your
app's Info.plist.

## Use

```swift
import Foundation
import OID4VCWallet

// Your Wallet Provider's backend, which attests the wallet and its keys
// after checking App Attest evidence.
struct MyWalletProvider: WalletProvider {
    func walletAttestation(clientID: String, instanceKey: Data) async throws -> String {
        fatalError("call your Wallet Provider")
    }
    func keyAttestation(keys: [Data], nonce: String) async throws -> String {
        fatalError("call your Wallet Provider")
    }
}

func example(offerLink: String, requestLink: String, pin: String) async throws {
    let keys = KeychainKeyStore()                  // Secure Enclave keys
    let wallet = try Wallet(
        configuration: WalletConfiguration(clientID: "my-wallet", redirectURI: "com.example.wallet:/callback"),
        keyStore: keys,
        credentialStore: try FileCredentialStore.standard(),
        provider: MyWalletProvider())
    _ = try await wallet.sweepOrphanedKeys(in: keys)   // at launch

    // Receive: an openid-credential-offer:// link.
    let issuance = try await wallet.startIssuance(offer: offerLink)
    if issuance.offer.grant == .preAuthorizedCode {
        try await issuance.redeemPreAuthorizedCode(pin: pin)
    } else {
        let url = try await issuance.beginAuthorization()
        // Open url in an ASWebAuthenticationSession, then:
        let redirect: URL = url
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

- [The OID4VCWallet documentation](https://idfoundry.github.io/OID4VCgo-wallet-swift/0.8.1/documentation/oid4vcwallet/): getting started, an article
  for each task, and the API reference.
- [OID4VCgo's `mobile/`](https://github.com/IDFoundry/OID4VCgo/tree/main/mobile):
  what each platform supports, in-person presentation, the Go side and
  its [ABI](https://github.com/IDFoundry/OID4VCgo/blob/main/mobile/ABI.md).
- [`mobile/MOBILE.md`](https://github.com/IDFoundry/OID4VCgo/blob/main/mobile/MOBILE.md):
  the design.
- [The demo wallet app](https://github.com/IDFoundry/OID4VCgo/tree/main/mobile/ios/DemoWallet):
  a complete app on this package, including the document provider.

## Contributing

This repository is generated: OID4VCgo's `wallet-swift-release`
workflow publishes every release here from
[`mobile/ios`](https://github.com/IDFoundry/OID4VCgo/tree/main/mobile/ios),
README included. Issues and pull requests belong in
[OID4VCgo](https://github.com/IDFoundry/OID4VCgo).

## License

MIT. See [LICENSE](LICENSE).

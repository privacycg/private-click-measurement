# Motivation

PCM’s conversion reports carry no cookies or click/user/browser identifying information. This is by design since a conversion should not be attributable to a specific click, user, or browser. This means there is no way for the server receiving the conversion report to tell if the report is trustworthy. The report may not even come from a browser since it’s just a stand-alone, stateless HTTP request. Ergo, a fraudster can submit reports in order to corrupt conversion measurement.

We want to allow cryptographic signatures to be included in attribution reports to convey their trustworthiness and prevent the kind of fraud mentioned above while not linking a specific user's activity across the two sites.

# Algorithm

This algorithm is implemented in WebKit, dependent on an underlying crypto framework, and matching what was proposed at the [Privacy CG meeting](https://github.com/privacycg/meetings/blob/main/2020/05-virtual/05-14-minutes.md#select-a-fraud-prevention-mechanism), May 14th, 2020.

1. The click source provides a source nonce in the clicked link using an attribute. The purpose of this source nonce is for the browser to be able to communicate with the click source server after the user has left the click source page and convey context of what the communication is about. In other words, sending the source nonce back to the click source server in a request tells the click source exactly which click the request is about, not just which user or browser. Such a link with relevant attributes looks like this:
   `<a attributionsourceid=3 attributiondestination="https://destination.example" attributionsourcenonce="ABCDEFabcdef0123456789">Link</a>`
2. The browser fetches the click source’s public key from `https://clicksource.example/.well-known/private-click-measurement/get-token-public-key/`. The response body looks like this:

```
    {
      "token_public_key": …
    }
```

3. The browser generates an unlinkable token.

4. The browser sends the unlinkable token together with the source nonce to the click source at `https://clicksource.example/.well-known/private-click-measurement/sign-unlinkable-token/`. The request body looks like this:

```
    {
      "source_engagement_type": "click",
      "source_nonce": …,
      "source_unlinkable_token": …,
      "version": 2
    }
```

5. The click source server signs the unlinkable token using RSA Blind Signature Scheme with Appendix (RSABSSA), proposed in the [IETF](https://datatracker.ietf.org/doc/draft-wood-cfrg-rsa-blind-signatures/).

6. The click source responds with the blinded signature token. The response body looks like this:

```
    {
      "unlinkable_token": …
    }
```

7. The browser generates a secret token for which the click source’s signature is valid but there is nothing linking it to the unlinkable token.

8. The triggering event happens.

9. The 24 to 48 hour delay passes.

10. The browser again fetches the click source’s public key from `https://clicksource.example/.well-known/private-click-measurement/get-token-public-key/`. It’s important that the key is fetched again since this is the defense against personalized signatures. The click source is not supposed to be able to re-identify the browser between these two events and it’s the browser’s job to uphold this protection. If the click source is able to re-identify the browser between the two fetches of its public key, it already has the ability to track the user across the events and nothing has been made worse by the potentially personalized signature.

11. The browser validates that the newly fetched public key is the same that was used to generate the unlinkable token.

12. The browser sends the attribution report to `https://clicksource.example/.well-known/private-click-measurement/report-attribution/` and `https://clickdestination.example/.well-known/private-click-measurement/report-attribution/` with the secret token and its signature. The request body looks like this:

```
    {
      "source_engagement_type": "click",
      "source_site": …,
      "source_id": …,
      "attributed_on_site": …,
      "trigger_data": …,
      "version": 2,
      "source_secret_token": …,
      "source_secret_token_signature": …
    }
```

13. The click source and the click destination validate the secret token to convince themselves that the click source deemed the click trustworthy when it happened. Note that he click destination needs to fetch the click source’s public key to validate the secret token and they need to store the public key if they want to validate later in time since there is no guarantee that the same public key will remain at the well-known location.

# Why Blinded Signatures?

PCM will send attribution reports to both source and destination sites and the full report should make sense to both parties. We want both parties to be able to validate the signature of secret tokens to check the authenticity of the report.

# Tokens for Attribution Destination Site Too?

We want to explore how to allow the destination site to also sign a token and thus provide proof of a trustworthy triggering event. Our current proposal is to combine this capability with the [proposed same-site pixel "API."](https://github.com/privacycg/private-click-measurement/issues/71) As you can see in the report structure, tokens and their signatures are prefixed with "source" so that we can have ones for the destination site too.
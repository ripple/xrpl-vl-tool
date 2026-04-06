# XRPL VL Signing Cheatsheet

## Step 1: Decode Current VL to an Editable File

```bash
VL_URL="https://vl.ripple.com/"
VL_TOOL="./xrpl_vl_tool"  # set to the actual path of the binary

# Save publisher manifest for later
curl -s $VL_URL | jq -r .manifest > publisher_manifest.txt

# Dump annotated manifests file (domain + nH ID as comment above each manifest)
curl -s $VL_URL | jq -r '.blob' | base64 --decode \
  | jq -r '.validators[] | "\(.validation_public_key)\t\(.manifest)"' \
  | while IFS=$'\t' read -r key manifest; do
    OUTPUT=$($VL_TOOL decode-manifest "$manifest" 2>&1)
    DOMAIN=$(echo "$OUTPUT" | grep 'Some("' | sed 's/.*Some("//;s/").*//')
    NH_ID=$(echo "$OUTPUT" | grep "Master Public Key:" | awk '{print $NF}')
    echo "# ${DOMAIN:-$key} ${NH_ID}"
    echo "$manifest"
  done > manifests.txt

# Next sequence (current + 1)
NEXT_SEQUENCE=$(($(curl -s $VL_URL | jq -r '.blob' | base64 --decode | jq '.sequence') + 1))
echo "Next sequence: $NEXT_SEQUENCE"
```

Output looks like:
```
# xrp.vet nHBWa56Vr7csoFcCnEPzCCKVvnDQw3L28mATgHYQMGtbEfUjuYyB
JAAAAAFxIe0Tqvy2qHvLXQk8LvN/BEMcKR...
# ED4246AA3AE9... nHBidG3pZK11zQD6kpNDoAhDxH6WLGui6ZxSbUx7LSqLHsgzMPec
JAAAAAFxIe1CRqo66dKYY5RIAMypGCnkRH...
# bithomp.com nHB8QMKGt9VB4Vg71VszjBVQnDW3v3QudM4DwFaJfy96bj4Pv9fA
JAAAAAJxIe04sCiOokC0zewYoaYonrSQB+...
```

Validators without a domain show their hex public key instead. The `nH...` ID matches what XRPLF PRs use.

## Step 2: Edit manifests.txt

Open `manifests.txt` in your editor.

**To remove:** delete the comment line and the manifest line below it.

**To add:** fetch the manifest from the XRPL network and append:

```bash
curl -s -X POST https://s1.ripple.com:51234 \
  -H "Content-Type: application/json" \
  -d '{"method":"manifest","params":[{"public_key":"<nH_PUBLIC_KEY>"}]}' \
  | jq -r '.result | "\(.details.domain) — \(.details.master_key)\n\(.manifest)"'
```

Paste the manifest line into `manifests.txt` with a `#` comment above it.

## Step 3: Sign

```bash
# Required for Vault secret provider (token is in 1Password)
export VAULT_TOKEN=''
export VAULT_ENDPOINT='https://vault-ui.mgt.ripplex.io/'

# Strip comments into a clean file
grep -v '^#' manifests.txt > manifests_clean.txt

$VL_TOOL sign \
  --vl-version 1 \
  --publisher-manifest "$(cat publisher_manifest.txt)" \
  --manifests-file manifests_clean.txt \
  --sequence $NEXT_SEQUENCE \
  --expiration 365 \
  --secret-provider vault \
  --secret-name unl-tool/mainnet:keypair
```

Secret providers: `local` (path to key file), `aws`, or `vault`.

## Step 4: Verify

```bash
$VL_TOOL load generated_vl_v1-*.json
```

## Step 4b: Diff against published VL

```bash
$VL_TOOL diff $VL_URL generated_vl_v1-*.json
```

Only changed fields (sequence, expiration, validator count, signatures) and added/removed validators are shown.

## Step 5: Publish

1. Rename the generated file to `index.json` and upload to the `vlmainnet` S3 bucket in the `xpring-xrpledger-prod` account.
2. Go to **CloudFront -> Distributions** and invalidate the appropriate distribution with `/*` as the object path.
3. Push the UNL file to the [ripple/vl](https://github.com/ripple/vl) repository.

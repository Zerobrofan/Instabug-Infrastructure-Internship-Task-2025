# Automating SealedSecrets Re-encryption with `kubeseal`  
**Infrastructure Internship Task 2025 – Solution Plan**  

## Objective  

Enhance the `kubeseal` CLI with an automated feature to re-encrypt all existing SealedSecrets in a Kubernetes cluster using the **latest active sealing key**, in order to support sealing key rotation and reduce long-term exposure to older keys.

## Summary of Requirements  

| Requirement | Notes |
|------------|-------|
| Identify all SealedSecrets in the cluster | Use `kubectl get sealedsecrets -A` |
| Fetch all active public keys | Already supported via controller |
| Decrypt using private key | Handled by controller, not exposed |
| Re-encrypt using latest public key | `kubeseal --re-encrypt` |
| Update SealedSecret objects | `kubectl apply -f` |
| Bonus: Logging, batching, security | Included below |

## Proposed CLI Command  

```bash
kubeseal rotate-all \
  --namespace <namespace> \
  [--all-namespaces] \
  [--selector <label-selector>] \
  [--output-dir ./rotated/] \
  [--apply] \
  [--report report.json]
```

## Directory Structure Example  

```bash
rotated/
├── mysecret-db.yaml
├── app-credentials.yaml
report.json
```

## Implementation Plan  

### 1. **List All SealedSecrets**
Use `kubectl` via Go client to list SealedSecrets:

```bash
kubectl get sealedsecrets -A -o json
```

Filter optionally via:
- `--namespace`
- `--selector`

### 2. **Dump Each SealedSecret to File**
Each retrieved SealedSecret is dumped to a temporary file in YAML/JSON format:

```bash
kubectl get sealedsecret <name> -n <ns> -o yaml > ./tmp/<name>.yaml
```

### 3. **Re-encrypt Each File with `kubeseal --re-encrypt`**

Use the existing CLI feature:

```bash
kubeseal --re-encrypt -f ./tmp/<name>.yaml -w ./rotated/<name>.yaml
```

> `--re-encrypt` leverages the SealedSecrets controller via the Kubernetes API. It does not expose secrets to the client and uses the latest active key.

This ensures:
- Secrets are never decrypted on the client
- Only the encryption key is rotated

### 4. **Apply the Re-encrypted SealedSecrets (optional)**

If `--apply` flag is passed, update in-cluster SealedSecrets:

```bash
kubectl apply -f ./rotated/<name>.yaml
```

This replaces the old encrypted payload with one encrypted by the newest key.

### 5. **(Bonus) Logging and Reporting**

While re-encrypting, collect metadata per file:

```json
{
  "name": "mysecret",
  "namespace": "prod",
  "oldKeyVersion": "v1",
  "newKeyVersion": "v3",
  "status": "success"
}
```

Output written to `report.json` or printed to console.

### 6. **(Bonus) Large-scale Support**

To support clusters with hundreds/thousands of SealedSecrets:
- Process in **parallel batches**
- Optionally support `--concurrency=N`
- Output progress bar or summary
- Use Go routines under the hood to run re-encryption in parallel

## Example Usage  

```bash
kubeseal rotate-all --all-namespaces --output-dir ./rotated --apply --report rotate-report.json
```

- Re-encrypts all SealedSecrets in all namespaces
- Writes updated files to `./rotated/`
- Applies changes in cluster
- Generates `rotate-report.json`

## Building on Existing Code  

This plan **requires no changes** to the encryption logic in `kubeseal` or the controller.

It builds on:

- `kubeseal --re-encrypt` (already available)
- Kubernetes API to list and apply SealedSecrets
- Local file system for staging

This feature would primarily involve:
- Adding a **new subcommand** hook to the CLI entry point in `cmd/kubeseal/main.go` using **Cobra**
- Using Go client-go libraries for `kubectl`-like functionality (e.g. listing all **SealedSecrets**)
- Re-using `kubeseal`'s own internal logic, much like the `--re-encrypt` implementation

## Testing Strategy  

1. Create a namespace with test SealedSecrets.
2. Run `kubeseal rotate-all --dry-run` to check output.
3. Run `kubeseal rotate-all --apply`.
4. Compare checksum or key version metadata of updated SealedSecrets.
5. Validate Secrets still decrypt and deploy correctly.

## Security Considerations  

- **Decryption handled securely** in-cluster by the controller.
- `kubeseal --re-encrypt` never exposes private key or plaintext.
- Temporary files stored locally for re-encryption are not secrets, only SealedSecrets.

## Notes  

- This process **does not eliminate old SealedSecrets** from Git or storage. That’s up to the user.
- It also **does not rotate the underlying secret value** — only its encryption key.
- Old keys are still valid for decryption unless explicitly removed.

## Outcome  

The `rotate-all` feature provides an **easy**, **secure**, and **scalable** way to re-encrypt SealedSecrets using the latest sealing key — maintaining key hygiene and reducing attack surface in environments using GitOps or public repositories.

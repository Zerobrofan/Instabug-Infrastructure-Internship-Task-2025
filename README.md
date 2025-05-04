# Automating SealedSecrets Re-encryption with kubeseal

This solution aims to extend the kubeseal CLI with a new command that automates the process of re-encrypting ***SealedSecrets*** in a **K8s** cluster with the latest key pair. The workflow should go as follows: ***identify all SealedSecrets, decrypt them using their respective keys, re-encrypt them with the latest key pair, and then update them in the cluster.***

## Implementation Plan

### 1. Command Structure
We will add a new command to kubeseal CLI called `rotate-secrets`:
```bash
kubeseal rotate-secrets [flags]
```

#### Flags:
- `--namespace`, `-n`: Specify namespace(s) to process
- `--selector`, `-l`: Label selector to filter SealedSecrets
- `--batch-size`: Number of secrets to process in parallel
- `--output-format`, `-o`: Output format for logs (json, yaml, text)
- `--backup`: Create backups before modifying SealedSecrets

### 2. Key Components

#### A. Secret Discovery

The first step is to identify all SealedSecrets in the cluster:
```go
func discoverSealedSecrets(namespace string, labelSelector string) ([]SealedSecret, error) {
    // Use the Kubernetes API to list all SealedSecrets in the specified namespace(s)
    // Filter results by the provided label selector if needed
}
```

This component will use the Kubernetes API to list all SealedSecrets in the specified namespace(s), with optional label filtering.

#### B. Key Management

The tool needs to interact with the sealed-secrets controller to access encryption keys:
```go
func fetchPublicKeys() ([]PublicKey, error) {
    // Retrieve all active public keys from the sealed-secrets controller
}

func identifyLatestKey(keys []PublicKey) (PublicKey, error) {
    // Identify the latest public key based on creation timestamp
}
```

#### C. Secret Processing

The core functionality involves decrypting and re-encrypting each SealedSecret:
```go
func processSecret(sealedSecret SealedSecret, latestKey PublicKey) (SealedSecret, error) {
    // Extract the encrypted data from the SealedSecret
    // Decrypt the data using the controller's private key
    // Re-encrypt the data using the latest public key
    // Create a new SealedSecret object with the re-encrypted data
    // Return the updated SealedSecret
}
```

#### D. Secret Update

Once re-encrypted, the SealedSecrets need to be updated in the cluster:
```go
func updateSealedSecret(originalSecret SealedSecret, updatedSecret SealedSecret) error {
    // Back up the original SealedSecret if backup flag is enabled
    // Update the SealedSecret in the cluster with the re-encrypted version
    // Verify the update was successful
    // Return any errors that occurred during the update
}
```

#### E. Logging and Reporting

A comprehensive logging system will track the re-encryption process:
```go
func initializeLogger(outputFormat string) (Logger, error) {
    // Set up logging based on the specified output format
}

func generateReport(results []ProcessingResult) (Report, error) {
    // Compile the results of the re-encryption process
    // Generate a summary report
}
```

### 3. Workflow Sequence

1. **Command Invocation**: User runs `kubeseal rotate-secrets` with desired parameters
2. **Authentication and Permission Check**: Verify user has sufficient permissions
3. **Discovery**: Identify all SealedSecrets in the specified scope
4. **Key Retrieval**: Fetch all active public keys from the controller
5. **Batch Processing**:
   - Divide SealedSecrets into batches for efficient processing
   - For each batch:
     - Process secrets in parallel
     - Log results and errors
6. **Reporting**: Generate and display a summary report

## Implementation Considerations

### Security Considerations

1. **Private Key Security**: The implementation must ensure that private keys never leave the Kubernetes cluster. The decryption process should be performed by the controller within the cluster.
2. **Access Control**: The tool should respect Kubernetes RBAC (Role-Based Access Control) and only allow authorized users to perform the re-encryption operation.
3. **Backup and Recovery**: Before modifying any SealedSecret, the tool should create backups to allow recovery in case of failures.
4. **Transactional Operations**: Changes should be atomic and reversible where possible to prevent partial updates that could lead to inconsistent states.

### Performance Considerations

1. **Batching**: Process SealedSecrets in batches to avoid overwhelming the Kubernetes API server and to improve performance.
2. **Parallelism**: Use concurrent processing to speed up the re-encryption of multiple SealedSecrets.
3. **Resource Constraints**: Consider cluster resource limitations and adjust batch sizes and parallelism accordingly.
4. **Incremental Processing**: Support for resuming the process in case of interruptions.

### Compatibility Considerations

1. **API Version Compatibility**: Ensure compatibility with different versions of the Kubernetes API and the sealed-secrets controller.
2. **Custom Resource Definition Updates**: Handle potential changes in the SealedSecret CRD structure.
3. **Backward Compatibility**: Maintain compatibility with existing SealedSecrets created with older versions of kubeseal.

## Error Handling

The implementation should include robust error handling for scenarios such as:
1. **Network Failures**: Handle temporary network issues with retries.
2. **Permission Errors**: Provide clear feedback when permissions are insufficient.
3. **Resource Not Found**: Handle cases where SealedSecrets may have been deleted during processing.
4. **Controller Unavailability**: Gracefully handle situations where the sealed-secrets controller is unavailable.
5. **Concurrency Issues**: Manage potential race conditions when multiple processes try to update the same SealedSecret.

## Documentation

### Basic Usage

To re-encrypt all SealedSecrets in a cluster:
```bash
kubeseal rotate-secrets
```

### Limiting to Specific Namespaces
```bash
kubeseal rotate-secrets --namespace=my-namespace
```

### Using Label Selectors
```bash
kubeseal rotate-secrets --selector="app=myapp"
```

### Adjusting Batch Size
```bash
kubeseal rotate-secrets --batch-size=5
```

### Disabling Backups
```bash
kubeseal rotate-secrets --backup=false
```

### Changing Output Format
```bash
kubeseal rotate-secrets --output-format=json
```

### Example Use Cases

1. **Regular Key Rotation**: Scheduled job to ensure all secrets use the latest key.
```bash
# Create a CronJob to run every month
kubectl create cronjob rotate-sealed-secrets --schedule="0 0 1 * *" --image=kubeseal -- rotate-secrets
```

2. **Before/After Cluster Upgrades**: Ensure all secrets are using the latest key before upgrading the cluster.
```bash
# Before upgrading
kubeseal rotate-secrets --output-format=json > rotation-report.json
```

3. **Selective Re-encryption**: Only re-encrypt secrets for specific applications.
```bash
kubeseal rotate-secrets --selector="app=frontend" --namespace=production
```

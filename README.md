# Automating SealedSecrets Re-encryption with kubeseal

The solution plans to add an additional command to the **kubeseal CLI** for re-encrypting **SealedSecrets within** a **K8s** cluster with the new key pair. The process should proceed as follows: ***find all existing SealedSecrets, decrypt them using their corresponding keys, re-encrypt them with the new key pair, and finally update them within the cluster.***

## Implementation Plan

### 1. Command Structure
We will introduce a new command for kubeseal CLI, named `rotate-secrets`:
```bash
kubeseal rotate-secrets [flags]
```

#### Flags:
- `--namespace`, `-n`: Name(s) of namespaces to be processed (default: all namespaces)
- `--selector`, `-l`: Label selector for filtering SealedSecrets
- `--batch-size`: The number of secrets to be processed in parallel (default is 10)
- `--output-format`, `-o`: File output format for generated logs (json, yaml, text)
- `--backup`: Create backups before modifying SealedSecrets

### 2. Key Components

#### A. Secret Discovery

The initial step is to find all the SealedSecrets within the cluster:
```go
func discoverSealedSecrets(namespace string, labelSelector string) ([]SealedSecret, error) {
    // Use the K8s API to list all the SealedSecrets in the given namespace(s)
    // Filter results according to the given label selector if necessary
}
```

This will utilize the K8s API to list all of the SealedSecrets in the given namespace(s), with optional label filtering.

#### B. Key Management

The tool must communicate with the sealed-secrets controller in order to obtain the encryption keys:
```go
func fetchPublicKeys() ([]PublicKey, error) {
    // Fetch all active public keys from the sealed-secrets controller
}

func identifyLatestKey(keys []PublicKey) (PublicKey, error) {
    // Identify the latest public key based on creation timestamp
}
```

#### C. Secret Processing

The core functionality involves decrypting and re-encrypting every SealedSecret:
```go
func processSecret(sealedSecret SealedSecret, latestKey PublicKey) (SealedSecret, error) {
    // Extract the encrypted data from the SealedSecret
    // Decrypt the data with the controller's private key
    // Re-encrypt the data using the latest public key
    // Create a new SealedSecret with the re-encrypted data
    // Return the updated SealedSecret
}
```

#### D. Secret Update

After being re-encrypted, the SealedSecrets are refreshed in the cluster to reflect the changes:
```go
func updateSealedSecret(originalSecret SealedSecret, updatedSecret SealedSecret) error {
    // Back up original SealedSecret when backup flag is turned on
    // Update the cluster's SealedSecret with the newly re-encrypted version
    // Ensure the update succeeded
    // Return the errors that happened when the update took place
}
```

#### E. Logging and Reporting

A simple logging system will track the re-encryption process:
```go
func initializeLogger(outputFormat string) (Logger, error) {
    // Set up logging based on the specified output format
}

func generateReport(results []ProcessingResult) (Report, error) {
    // Compile the outcomes of the re-encryption process
    // Generate a summary report
}
```

### 3. Workflow Sequence

1. **Method Invocation**: The user invokes `kubeseal rotate-secrets` with desired parameters
2. **Authentication and Permissions**: Ensure user is within their permission bounds to access the encryption keys.
3. **Discovery**: Find all SealedSecrets within the given scope
4. **Key Retrieval**: Retrieve all active public keys from the controller
5. **Batch Processing**:
   - Separate SealedSecrets in batches for processing efficiently
   - For every batch:
     - Process secrets in parallel
     - Record results and errors
6. **Reporting**: Create and show a summary report

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

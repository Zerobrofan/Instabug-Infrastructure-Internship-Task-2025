# Automating SealedSecrets Re-encryption with kubeseal

This solution plans to add an additional command to the **kubeseal CLI** for re-encrypting **SealedSecrets within** a **K8s** cluster with the new key pair. The process should proceed as follows: ***find all existing SealedSecrets, decrypt them using their corresponding keys, re-encrypt them with the new key pair, and finally update them within the cluster*** <br><br>
**🛑 NOTE: Any code snippets written here are written in GO**

## Implementation Plan

### 1. Command Structure
We will introduce a new command for kubeseal CLI, named `rotate-secrets`:
```bash
kubeseal rotate-secrets [flags]
```

#### Flags:
- `--namespace`, `-n`: Name(s) of namespaces to be procesed (default: all namespaces)
- `--selector`, `-l`: Label selector for filtering SealedSecrets
- `--batch-size`: The number of secrets to be processed in parallel (default is 10)
- `--output-format`, `-o`: File output format for generated log (json, yaml, text)
- `--backup`: Create backups before modfying SealedSecrets

### 2. Key Components

#### A. Secret Discovery

The initial step is to find all the SealedSecrets within the cluster:
```go
func discoverSealedSecrets(namespace string, labelSelector string) ([]SealedSecret, error) {
    // use the K8s API to list all the SealedSecrets in the given namespace(s)
    // filter results according to the given label selector if necessary
}
```

This will utilise the K8s API to list all of the SealedSecrets in the given namespace(s), with optional label filtring

#### B. Key Management

The tool must communicate with the sealed-secrets controller in order to obtain the encryption keys:
```go
func fetchPublicKeys() ([]PublicKey, error) {
    // fetch the currently actve public keys from the sealed-secrets controller
}

func identifyLatestKey(keys []PublicKey) (PublicKey, error) {
    // identify the latest public key based on creation timestamp
}
```

#### C. Secret Processing

The core functionality involves decrypting and re-encrypting every SealedSecret:
```go
func processSecret(sealedSecret SealedSecret, latestKey PublicKey) (SealedSecret, error) {
    // extract the encrypted data from the SealedSecret
    // decrypt the data with the controller's private key
    // re-encrypt the data using the latst public key
    // create a new SealedSecret with the re-encrypted data
    // return the updated SealedSecret
}
```

#### D. Secret Update

After being re-encrypted, the SealedSecrets are refreshed in the cluster to reflect the changes:
```go
func updateSealedSecret(originalSecret SealedSecret, updatedSecret SealedSecret) error {
    // back up original SealedSecret when backup flag is turned on
    // update the cluster's SealedSecret with the newly re-encrypted version
    // ensure the update succeeded
    // return the errors that happened when the update took place
}
```

#### E. Logging and Reporting

A simple logging system will track the re-encryption process:
```go
func initializeLogger(outputFormat string) (Logger, error) {
    // set up logging based on the specfied output format
}

func generateReport(results []ProcessingResult) (Report, error) {
    // compile the outcomes of the re-encryption process
    // generate a summary report
}
```

### 3. Workflow Sequence

1. **Method Invocation**: The user invokes `kubeseal rotate-secrets` with desired parametrs
2. **Authentication and Permissions**: Ensure user is within their permssion bounds to access the encryption keys.
3. **Discovery**: Find all SealedSecrets within the given scope
4. **Key Retrieval**: Retrieve all active public keys from the controller
5. **Batch Processing**:
   - Separate SealedSecrets in batches for processing effciently
   - For every batch:
     - Process secrets in parallel
     - Record results and errors
6. **Reporting**: Create and show a summary report

## Implementation Considerations

### Security Considerations

1. **Private Key Security**: The solution should guarantee private keys are not exported out of the K8s cluster. The controller should decrypt in-cluster instead
2. **Access Control**: The tool must respect K8s RBAC (Role-Based Access Control) and only permit users with permision to carry out the re-encryption action
3. **Backup and Recovery**: The tool should make backups prior to changing any SealedSecret in order for recovery in the event of failure
4. **Transactional Operations**: Transactions must be atomic and reversble where appropriate in order to avoid partial updates that will create inconistencies

### Performance Considerations

1. **Batching**: Process SealedSecrets in batches so that the K8s API server is not overwhelmed and performence is optimized
2. **Parallelism**: Utilize concurrent processing in order to accelerate the re-encryption of several SealedSecrets
3. **Resource Constraints**: Account for cluster resource constraints and scale batch sizes and paralelism accordingly
4. **Incremental Processing**: Support resuming the process in the event of interuptions

### Compatibility Considerations

1. **API Version Compatibility**: Maintain compatiblity with various API versions of K8s as well as the sealed-secrets controller
2. **Custom Resource Definition Updates**: Handle potential changes in the structure of the SealedSecret CRD
3. **Backward Compatibility**: Preserve compatibilty with existing SealedSecrets generated with previous releases of kubeseal

## Error Handling

The implemntation must have error handling in place for situations like:
1. **Network Failures**: Recover from temporory network failure with retries
2. **Permission Errors**: Provide informative feedback when permssions are lacking
3. **Resource Not Found**: Handle situtions in which SealedSecrets could have been deleted
4. **Controller Unavailability**: Handle with cases when the sealed-secrets controller is not availabe
5. **Concurrency Issues**: Adress potential race condtions when several proceses attempt to update the same SealedSecret

## Documentation

### Example Usage

#### Re-encrypting all SealedSecrets in a cluster
```bash
kubeseal rotate-secrets
```

#### Limiting to Specific Namespaces
```bash
kubeseal rotate-secrets --namespace=my-namespace
```

#### Using Label Selectors
```bash
kubeseal rotate-secrets --selector="app=myapp"
```

#### Adjusting Batch Size
```bash
kubeseal rotate-secrets --batch-size=5
```

#### Disabling Backups
```bash
kubeseal rotate-secrets --backup=false
```

#### Changing Output Format
```bash
kubeseal rotate-secrets --output-format=json
```

### Example Use Cases

1. **Regular Key Rotation**: Schduled job to ensure all secrets use the latest key.
```bash
# create a cronjob to run every month
kubectl create cronjob rotate-sealed-secrets --schedule="0 0 1 * *" --image=kubeseal -- rotate-secrets
```

2. **Before/After Cluster Upgrades**: Ensure all secrets are using the latest key before upgrading the cluster.
```bash
# before upgrading
kubeseal rotate-secrets --output-format=json > rotation-report.json
```

3. **Selective Re-encryption**: Only re-encrypt secrets for specific applications.
```bash
kubeseal rotate-secrets --selector="app=frontend" --namespace=production
```

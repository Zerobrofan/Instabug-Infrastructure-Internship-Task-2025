# Automating SealedSecrets Re-encryption in Kubernetes

## Introduction

This document outlines a proposed implementation for automating the re-encryption of SealedSecrets in Kubernetes clusters as a new feature for the kubeseal CLI tool. The current mechanism requires manual re-encryption of existing SealedSecrets when key rotation occurs, which can be time-consuming and error-prone, especially in environments with many secrets. This implementation aims to streamline this process by providing an automated solution.

## Background

Bitnami's sealed-secrets controller (kubeseal) is a valuable tool that allows users to encrypt Kubernetes secrets, enabling secure storage in version control systems. The controller performs key rotation every 30 days by default, generating new key pairs. However, existing SealedSecrets continue to use older keys, and converting them to use the new key requires manual re-encryption.

## Design Overview

The proposed solution will extend the kubeseal CLI with a new command that automates the process of re-encrypting all SealedSecrets in a Kubernetes cluster using the latest encryption key. This will involve identifying all SealedSecrets, decrypting them using their respective keys, re-encrypting them with the latest key, and updating them in the cluster.

## Implementation Plan

### 1. Command Structure

We will add a new command to kubeseal CLI called `rotate-secrets`:

```bash
kubeseal rotate-secrets [flags]
```

#### Flags:
- `--namespace`, `-n`: Specify namespace(s) to process (default: all namespaces)
- `--selector`, `-l`: Label selector to filter SealedSecrets
- `--dry-run`: Simulate the process without making changes
- `--batch-size`: Number of secrets to process in parallel (default: 10)
- `--timeout`: Timeout for operations (default: 5m)
- `--output-format`, `-o`: Output format for logs (json, yaml, text)
- `--verbose`, `-v`: Enable verbose logging
- `--backup`: Create backups before modifying SealedSecrets

### 2. Key Components

#### A. Secret Discovery

The first step is to identify all SealedSecrets in the cluster:

```go
func discoverSealedSecrets(namespace string, labelSelector string) ([]SealedSecret, error) {
    // Use the Kubernetes API to list all SealedSecrets in the specified namespace(s)
    // Filter results by the provided label selector if specified
    // Return the list of discovered SealedSecrets
}
```

This component will use the Kubernetes API to list all SealedSecrets in the specified namespace(s), with optional filtering based on labels.

#### B. Key Management

The tool needs to interact with the sealed-secrets controller to access encryption keys:

```go
func fetchPublicKeys() ([]PublicKey, error) {
    // Retrieve all active public keys from the sealed-secrets controller
    // Return the list of public keys with their respective metadata
}

func identifyLatestKey(keys []PublicKey) (PublicKey, error) {
    // Identify the latest public key based on creation timestamp
    // Return the latest key to be used for re-encryption
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
func initializeLogger(outputFormat string, verbose bool) (Logger, error) {
    // Set up logging based on the specified output format and verbosity
    // Return a configured logger
}

func generateReport(results []ProcessingResult) (Report, error) {
    // Compile the results of the re-encryption process
    // Generate a summary report
    // Return the formatted report
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
7. **Cleanup**: Remove any temporary resources

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

## Detailed Implementation Steps

### Step 1: Extension of the CLI Command Structure

Extend the kubeseal CLI command structure to include the new `rotate-secrets` command:

```go
func NewRotateSecretsCommand() *cobra.Command {
    cmd := &cobra.Command{
        Use:   "rotate-secrets",
        Short: "Re-encrypt all SealedSecrets using the latest key",
        Long:  `This command identifies all SealedSecrets in the cluster, decrypts them, and re-encrypts them using the latest key.`,
        RunE: func(cmd *cobra.Command, args []string) error {
            return executeRotateSecrets(cmd, args)
        },
    }

    // Add flags
    cmd.Flags().StringP("namespace", "n", "", "Namespace to process (default: all namespaces)")
    cmd.Flags().StringP("selector", "l", "", "Label selector to filter SealedSecrets")
    cmd.Flags().Bool("dry-run", false, "Simulate the process without making changes")
    cmd.Flags().Int("batch-size", 10, "Number of secrets to process in parallel")
    cmd.Flags().Duration("timeout", 5*time.Minute, "Timeout for operations")
    cmd.Flags().StringP("output-format", "o", "text", "Output format for logs (json, yaml, text)")
    cmd.Flags().BoolP("verbose", "v", false, "Enable verbose logging")
    cmd.Flags().Bool("backup", true, "Create backups before modifying SealedSecrets")

    return cmd
}
```

### Step 2: SealedSecret Discovery Implementation

Implement the discovery of SealedSecrets in the cluster:

```go
func discoverSealedSecrets(clientset kubernetes.Interface, namespace string, labelSelector string) ([]ssv1alpha1.SealedSecret, error) {
    sealedSecretClient, err := getSealedSecretClient(clientset)
    if err != nil {
        return nil, fmt.Errorf("failed to create SealedSecret client: %w", err)
    }

    var namespaces []string
    if namespace != "" {
        namespaces = []string{namespace}
    } else {
        // List all namespaces
        nsList, err := clientset.CoreV1().Namespaces().List(context.TODO(), metav1.ListOptions{})
        if err != nil {
            return nil, fmt.Errorf("failed to list namespaces: %w", err)
        }
        for _, ns := range nsList.Items {
            namespaces = append(namespaces, ns.Name)
        }
    }

    var sealedSecrets []ssv1alpha1.SealedSecret
    for _, ns := range namespaces {
        listOptions := metav1.ListOptions{}
        if labelSelector != "" {
            listOptions.LabelSelector = labelSelector
        }

        secretList, err := sealedSecretClient.SealedSecrets(ns).List(context.TODO(), listOptions)
        if err != nil {
            return nil, fmt.Errorf("failed to list SealedSecrets in namespace %s: %w", ns, err)
        }

        sealedSecrets = append(sealedSecrets, secretList.Items...)
    }

    return sealedSecrets, nil
}
```

### Step 3: Key Management Implementation

Implement the fetching of public keys from the sealed-secrets controller:

```go
func fetchPublicKeys(clientset kubernetes.Interface) ([]PublicKey, error) {
    // Get the sealed-secrets controller pod
    pods, err := clientset.CoreV1().Pods("kube-system").List(context.TODO(), metav1.ListOptions{
        LabelSelector: "name=sealed-secrets-controller",
    })
    if err != nil {
        return nil, fmt.Errorf("failed to find sealed-secrets controller: %w", err)
    }
    if len(pods.Items) == 0 {
        return nil, fmt.Errorf("no sealed-secrets controller found")
    }

    // Request the public keys from the controller
    req := clientset.CoreV1().RESTClient().Get().
        Namespace("kube-system").
        Resource("services").
        Name("sealed-secrets-controller:8080").
        SubResource("proxy").
        Suffix("/v1/cert.pem")

    result := &bytes.Buffer{}
    err = req.Do(context.TODO()).Stream(context.TODO(), func(stream io.ReadCloser) error {
        _, err := io.Copy(result, stream)
        return err
    })
    if err != nil {
        return nil, fmt.Errorf("failed to fetch public key from controller: %w", err)
    }

    // Parse the public keys from the response
    block, _ := pem.Decode(result.Bytes())
    if block == nil {
        return nil, fmt.Errorf("failed to parse PEM block containing the public key")
    }

    cert, err := x509.ParseCertificate(block.Bytes)
    if err != nil {
        return nil, fmt.Errorf("failed to parse certificate: %w", err)
    }

    publicKey := PublicKey{
        Key:       cert.PublicKey,
        CreatedAt: cert.NotBefore,
        ExpiresAt: cert.NotAfter,
    }

    return []PublicKey{publicKey}, nil
}

func identifyLatestKey(keys []PublicKey) (PublicKey, error) {
    if len(keys) == 0 {
        return PublicKey{}, fmt.Errorf("no public keys available")
    }

    latestKey := keys[0]
    for _, key := range keys {
        if key.CreatedAt.After(latestKey.CreatedAt) {
            latestKey = key
        }
    }

    return latestKey, nil
}
```

### Step 4: Secret Processing Implementation

Implement the decryption and re-encryption of SealedSecrets:

```go
func processSecret(clientset kubernetes.Interface, sealedSecret ssv1alpha1.SealedSecret, latestKey PublicKey) (ssv1alpha1.SealedSecret, error) {
    // Create a temporary Kubernetes Secret object for decryption
    tempSecret := &corev1.Secret{
        ObjectMeta: metav1.ObjectMeta{
            Name:      sealedSecret.Name + "-temp",
            Namespace: sealedSecret.Namespace,
        },
    }

    // Use the sealed-secrets controller to decrypt the SealedSecret
    err := clientset.CoreV1().RESTClient().Post().
        Namespace("kube-system").
        Resource("services").
        Name("sealed-secrets-controller:8080").
        SubResource("proxy").
        Suffix("/v1/decrypt").
        Body(sealedSecret.Spec.EncryptedData).
        Do(context.TODO()).
        Into(tempSecret)
    if err != nil {
        return ssv1alpha1.SealedSecret{}, fmt.Errorf("failed to decrypt SealedSecret: %w", err)
    }

    // Re-encrypt the Secret using the latest key
    newSealedSecret, err := encryptSecret(clientset, tempSecret, latestKey)
    if err != nil {
        return ssv1alpha1.SealedSecret{}, fmt.Errorf("failed to re-encrypt Secret: %w", err)
    }

    // Clean up the temporary Secret
    err = clientset.CoreV1().Secrets(tempSecret.Namespace).Delete(context.TODO(), tempSecret.Name, metav1.DeleteOptions{})
    if err != nil {
        // Log the error but continue
        log.Printf("Warning: failed to delete temporary Secret: %v", err)
    }

    return newSealedSecret, nil
}

func encryptSecret(clientset kubernetes.Interface, secret *corev1.Secret, publicKey PublicKey) (ssv1alpha1.SealedSecret, error) {
    // Use the kubeseal API to encrypt the Secret
    req := clientset.CoreV1().RESTClient().Post().
        Namespace("kube-system").
        Resource("services").
        Name("sealed-secrets-controller:8080").
        SubResource("proxy").
        Suffix("/v1/seal").
        Body(secret)

    result := &ssv1alpha1.SealedSecret{}
    err := req.Do(context.TODO()).Into(result)
    if err != nil {
        return ssv1alpha1.SealedSecret{}, fmt.Errorf("failed to encrypt Secret: %w", err)
    }

    return *result, nil
}
```

### Step 5: Secret Update Implementation

Implement the update of SealedSecrets in the cluster:

```go
func updateSealedSecret(clientset kubernetes.Interface, originalSecret ssv1alpha1.SealedSecret, updatedSecret ssv1alpha1.SealedSecret, createBackup bool) error {
    sealedSecretClient, err := getSealedSecretClient(clientset)
    if err != nil {
        return fmt.Errorf("failed to create SealedSecret client: %w", err)
    }

    // Create a backup if requested
    if createBackup {
        backupName := originalSecret.Name + "-backup-" + time.Now().Format("20060102150405")
        backupSecret := originalSecret.DeepCopy()
        backupSecret.Name = backupName
        backupSecret.ResourceVersion = ""

        _, err := sealedSecretClient.SealedSecrets(originalSecret.Namespace).Create(context.TODO(), backupSecret, metav1.CreateOptions{})
        if err != nil {
            return fmt.Errorf("failed to create backup of SealedSecret: %w", err)
        }
    }

    // Update the original SealedSecret with the re-encrypted data
    updatedSecret.ResourceVersion = originalSecret.ResourceVersion
    _, err = sealedSecretClient.SealedSecrets(originalSecret.Namespace).Update(context.TODO(), &updatedSecret, metav1.UpdateOptions{})
    if err != nil {
        return fmt.Errorf("failed to update SealedSecret: %w", err)
    }

    return nil
}
```

### Step 6: Logging and Reporting Implementation

Implement logging and reporting:

```go
type Logger struct {
    outputFormat string
    verbose      bool
    writer       io.Writer
}

func initializeLogger(outputFormat string, verbose bool, writer io.Writer) (Logger, error) {
    if outputFormat != "text" && outputFormat != "json" && outputFormat != "yaml" {
        return Logger{}, fmt.Errorf("invalid output format: %s", outputFormat)
    }

    return Logger{
        outputFormat: outputFormat,
        verbose:      verbose,
        writer:       writer,
    }, nil
}

func (l Logger) Log(message string, level string, data interface{}) {
    if level == "debug" && !l.verbose {
        return
    }

    entry := map[string]interface{}{
        "timestamp": time.Now().Format(time.RFC3339),
        "level":     level,
        "message":   message,
    }

    if data != nil {
        entry["data"] = data
    }

    switch l.outputFormat {
    case "json":
        jsonData, _ := json.Marshal(entry)
        fmt.Fprintln(l.writer, string(jsonData))
    case "yaml":
        yamlData, _ := yaml.Marshal(entry)
        fmt.Fprintln(l.writer, string(yamlData))
    default:
        fmt.Fprintf(l.writer, "[%s] %s: %s\n", entry["timestamp"], strings.ToUpper(level), message)
        if data != nil {
            fmt.Fprintf(l.writer, "  Data: %v\n", data)
        }
    }
}

type ProcessingResult struct {
    SealedSecret ssv1alpha1.SealedSecret
    Success      bool
    Error        error
    Duration     time.Duration
}

type Report struct {
    StartTime        time.Time
    EndTime          time.Time
    TotalSecrets     int
    SuccessfulSecrets int
    FailedSecrets     int
    Errors           []string
}

func generateReport(results []ProcessingResult) Report {
    report := Report{
        TotalSecrets: len(results),
    }

    for _, result := range results {
        if result.Success {
            report.SuccessfulSecrets++
        } else {
            report.FailedSecrets++
            if result.Error != nil {
                report.Errors = append(report.Errors, fmt.Sprintf("%s/%s: %v", result.SealedSecret.Namespace, result.SealedSecret.Name, result.Error))
            }
        }
    }

    return report
}
```

### Step 7: Main Execution Flow

Implement the main execution flow:

```go
func executeRotateSecrets(cmd *cobra.Command, args []string) error {
    // Parse flags
    namespace, _ := cmd.Flags().GetString("namespace")
    labelSelector, _ := cmd.Flags().GetString("selector")
    dryRun, _ := cmd.Flags().GetBool("dry-run")
    batchSize, _ := cmd.Flags().GetInt("batch-size")
    timeout, _ := cmd.Flags().GetDuration("timeout")
    outputFormat, _ := cmd.Flags().GetString("output-format")
    verbose, _ := cmd.Flags().GetBool("verbose")
    createBackup, _ := cmd.Flags().GetBool("backup")

    // Initialize logger
    logger, err := initializeLogger(outputFormat, verbose, os.Stdout)
    if err != nil {
        return err
    }

    // Create Kubernetes client
    config, err := getKubeConfig()
    if err != nil {
        return fmt.Errorf("failed to get Kubernetes config: %w", err)
    }

    clientset, err := kubernetes.NewForConfig(config)
    if err != nil {
        return fmt.Errorf("failed to create Kubernetes client: %w", err)
    }

    // Set context with timeout
    ctx, cancel := context.WithTimeout(context.Background(), timeout)
    defer cancel()

    // Discover SealedSecrets
    logger.Log("Discovering SealedSecrets", "info", map[string]interface{}{
        "namespace":     namespace,
        "labelSelector": labelSelector,
    })

    sealedSecrets, err := discoverSealedSecrets(clientset, namespace, labelSelector)
    if err != nil {
        return fmt.Errorf("failed to discover SealedSecrets: %w", err)
    }

    logger.Log(fmt.Sprintf("Found %d SealedSecrets", len(sealedSecrets)), "info", nil)

    // Fetch public keys
    logger.Log("Fetching public keys from sealed-secrets controller", "info", nil)
    publicKeys, err := fetchPublicKeys(clientset)
    if err != nil {
        return fmt.Errorf("failed to fetch public keys: %w", err)
    }

    // Identify the latest key
    latestKey, err := identifyLatestKey(publicKeys)
    if err != nil {
        return fmt.Errorf("failed to identify latest key: %w", err)
    }

    logger.Log("Using latest public key for re-encryption", "info", map[string]interface{}{
        "createdAt": latestKey.CreatedAt,
        "expiresAt": latestKey.ExpiresAt,
    })

    // Process secrets in batches
    var results []ProcessingResult
    for i := 0; i < len(sealedSecrets); i += batchSize {
        end := i + batchSize
        if end > len(sealedSecrets) {
            end = len(sealedSecrets)
        }

        batch := sealedSecrets[i:end]
        logger.Log(fmt.Sprintf("Processing batch %d/%d", i/batchSize+1, (len(sealedSecrets)+batchSize-1)/batchSize), "info", nil)

        var wg sync.WaitGroup
        resultsChan := make(chan ProcessingResult, len(batch))

        for _, sealedSecret := range batch {
            wg.Add(1)
            go func(ss ssv1alpha1.SealedSecret) {
                defer wg.Done()

                startTime := time.Now()
                result := ProcessingResult{
                    SealedSecret: ss,
                    Success:      false,
                }

                logger.Log(fmt.Sprintf("Processing SealedSecret %s/%s", ss.Namespace, ss.Name), "debug", nil)

                // Process the secret
                updatedSecret, err := processSecret(clientset, ss, latestKey)
                if err != nil {
                    logger.Log(fmt.Sprintf("Failed to process SealedSecret %s/%s: %v", ss.Namespace, ss.Name, err), "error", nil)
                    result.Error = err
                    resultsChan <- result
                    return
                }

                // Skip update if dry run
                if dryRun {
                    logger.Log(fmt.Sprintf("Dry run: would update SealedSecret %s/%s", ss.Namespace, ss.Name), "info", nil)
                    result.Success = true
                } else {
                    // Update the secret
                    err = updateSealedSecret(clientset, ss, updatedSecret, createBackup)
                    if err != nil {
                        logger.Log(fmt.Sprintf("Failed to update SealedSecret %s/%s: %v", ss.Namespace, ss.Name, err), "error", nil)
                        result.Error = err
                        resultsChan <- result
                        return
                    }

                    logger.Log(fmt.Sprintf("Successfully updated SealedSecret %s/%s", ss.Namespace, ss.Name), "info", nil)
                    result.Success = true
                }

                result.Duration = time.Since(startTime)
                resultsChan <- result
            }(sealedSecret)
        }

        wg.Wait()
        close(resultsChan)

        for result := range resultsChan {
            results = append(results, result)
        }
    }

    // Generate and display report
    report := generateReport(results)
    logger.Log("Re-encryption process completed", "info", map[string]interface{}{
        "totalSecrets":      report.TotalSecrets,
        "successfulSecrets": report.SuccessfulSecrets,
        "failedSecrets":     report.FailedSecrets,
    })

    if report.FailedSecrets > 0 {
        logger.Log("Some secrets failed to re-encrypt", "warning", map[string]interface{}{
            "errors": report.Errors,
        })
    }

    return nil
}
```

## Usage Documentation

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

### Dry Run Mode

To simulate the process without making changes:

```bash
kubeseal rotate-secrets --dry-run
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

### Verbose Logging

```bash
kubeseal rotate-secrets --verbose
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
kubeseal rotate-secrets --verbose --output-format=json > rotation-report.json
```

3. **Selective Re-encryption**: Only re-encrypt secrets for specific applications.

```bash
kubeseal rotate-secrets --selector="app=frontend" --namespace=production
```

## Conclusion

This implementation plan provides a robust and efficient mechanism for automating the re-encryption of SealedSecrets in Kubernetes clusters. The proposed solution integrates seamlessly with the existing kubeseal CLI, enhancing its functionality without compromising security or usability.

The implementation addresses all the required features, including:
- Discovery of SealedSecrets across the cluster
- Fetching of public keys from the controller
- Secure decryption and re-encryption of secrets
- Efficient batch processing of large numbers of secrets
- Comprehensive logging and reporting
- Backup mechanisms for safety

Additionally, the bonus features are implemented:
- Detailed logging and reporting
- Efficient handling of large numbers of secrets through batching and parallelism
- Security measures to protect private keys

By implementing this feature, the kubeseal CLI will provide users with a streamlined process for keeping their SealedSecrets up-to-date with the latest encryption keys, enhancing both security and operational efficiency.

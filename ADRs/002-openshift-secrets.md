### Secrets Management in OpenShift using SOPS 
- **Status:** Accepted
- **Context:** Storing raw application passwords, certificates and private keys in ConfigMaps or code is insecure. OpenShift Secrets are designed for sensitive data but are only Base64 encoded by default, not encrypted at rest. The goal is to manage Kubernetes resources as code, including secrets, without exposing plaintext values in source control. The solution must integrate with our existing OpenShift and CI/CD Jenkins pipelines.
- **Decision:**
  1. **Encryption:** Use Mozilla SOPS to encrypt secrets (YAML/JSON) in Git, using PGP or AWS KMS/HashiCorp Vault for key management.
  2. **Storage:** Store encrypted secrets as kind: Secret in OpenShift.
  3. **Deployment:** Use Helm Charts to deploy the secrets using Jenkins.
  4. **Access Pattern:** Mount the secret/configmap to the Pod using a volumeMount and refererence it to Environment variable, which can be used in configMap to access the password
  5. **Persistence/Sync:** If dynamic updates are needed, use a PersistentVolumeClaim (PVC) to sync the secret file from the Secret object to a volume. 
- **Consequences:**
  1. **Positive:** Secrets are encrypted in Git (Security compliance). Application receives files, improving security over environment variables. No need to re-run pods when secret contents change (using volume mounts).
  2. **Negative:** Added complexity in CI/CD pipeline for decrypting/applying secrets. Requires management of SOPS keys. 

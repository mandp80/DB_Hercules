### Use OpenShift Secrets for Encrypted Password Storage 
- **Status:** Accepted
- **Context:** Storing raw application passwords, certificates and private keys in ConfigMaps or code is insecure. OpenShift Secrets are designed for sensitive data but are only Base64 encoded by default, not encrypted at rest. The goal is to manage Kubernetes resources as code, including secrets, without exposing plaintext values in source control. The solution must integrate with our existing OpenShift and CI/CD Jenkins pipelines.
- **Decision:** Use OpenShift Secrets to store encrypted passwords, specifically mounting them as files, combined with etcd encryption to protect data at rest.
- **Consequences:** Enhances security; requires managing encryption keys, and requires application updates to decrypt passwords at runtime.

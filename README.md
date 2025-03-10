# **Devtron CI Security Scanning with Dependency-Track, Syft, and Trivy**

## **Overview**
This repository provides an automated solution for container security scanning in **Devtron CI/CD**.  
It integrates **Syft** and **Trivy** to generate **Software Bill of Materials (SBOMs)** and **vulnerability reports**, which are then uploaded to **Dependency-Track** for monitoring.

---

## **1. Features**
✔️ **Automated SBOM and vulnerability scanning** after container image build  
✔️ **Uses Syft for SBOM generation**  
✔️ **Uses Trivy for vulnerability detection**  
✔️ **Uploads scan reports to Dependency-Track** via API  
✔️ **Runs in Devtron CI/CD pipelines**  

---

## **2. Prerequisites**
Ensure you have:
- **A running Kubernetes cluster**
- **Helm installed**
- **Dependency-Track deployed using Helm**
- **Devtron CI/CD configured**
- **Valid API Token & Project UUID for Dependency-Track**

---

## **3. Installation of Dependency-Track (Helm)**
```bash
helm repo add dependency-track https://dependencytrack.github.io/helm-charts
helm repo update
helm install dtrack dependency-track/dependency-track --namespace dtrack --create-namespace
```

Verify Deployment:
```bash
kubectl get pods -n dtrack
```

---

## **4. Setup in Devtron CI/CD**
### **4.1 Define Environment Variables**
Before running the script, configure these variables in **Devtron CI/CD**:

| Variable               | Description |
|------------------------|------------|
| `IMAGE_NAME`           | Container image name (e.g., `myrepo/app:latest`) |
| `SBOM_REPORT_FILE`     | Path to store the Syft SBOM file (`sbom.json`) |
| `TRIVY_REPORT_FILE`    | Path to store the Trivy scan file (`trivy.json`) |
| `TRIVY_CYCLONEDX_FILE` | Path to store Trivy CycloneDX SBOM (`trivy-cyclonedx.json`) |
| `DTRACK_API_URL`       | Dependency-Track API endpoint (e.g., `http://dtrack.example.com/api/v1/bom`) |
| `DTRACK_API_KEY`       | API Key for Dependency-Track |
| `PROJECT_UUID`         | Unique Project UUID in Dependency-Track |

---

## **5. Security Scanning Script**
### **5.1 Script for Devtron CI/CD**
Save the following script as `security_scan.sh` and integrate it into your **Devtron CI/CD pipeline**.

```sh
#!/bin/sh
set -eo pipefail
#set -v  ## Uncomment for debugging

# Function to install Syft if not installed
install_syft() {
    echo "Syft not found. Installing..."
    curl -sSfL https://raw.githubusercontent.com/anchore/syft/main/install.sh | sh -s -- -b /usr/local/bin
    echo "Syft installed successfully."
}

# Function to install Trivy if not installed
install_trivy() {
    echo "Trivy not found. Installing..."
    curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin
    echo "Trivy installed successfully."
}

# Check if Syft is installed; install if missing
if ! command -v syft &> /dev/null; then
    install_syft
else
    echo "Syft is already installed."
fi

# Check if Trivy is installed; install if missing
if ! command -v trivy &> /dev/null; then
    install_trivy
else
    echo "Trivy is already installed."
fi

# Run Syft scan to generate a CycloneDX SBOM
echo "Scanning the container image ($IMAGE_NAME) with Syft..."
if syft "$IMAGE_NAME" -o cyclonedx-json > "$SBOM_REPORT_FILE"; then
    echo "Syft scan complete. SBOM saved at $SBOM_REPORT_FILE."
else
    echo "Syft scan failed!" >&2
    exit 1
fi

# Run Trivy scan to generate a JSON vulnerability report
echo "Scanning the container image ($IMAGE_NAME) with Trivy..."
if trivy image --format json --output "$TRIVY_REPORT_FILE" "$IMAGE_NAME"; then
    echo "Trivy scan complete. Report saved at $TRIVY_REPORT_FILE."
else
    echo "Trivy scan failed!" >&2
    exit 1
fi

# Convert Trivy vulnerability report to CycloneDX format
echo "Converting Trivy report to CycloneDX format..."
if trivy image --format cyclonedx --output "$TRIVY_CYCLONEDX_FILE" "$IMAGE_NAME"; then
    echo "Trivy CycloneDX report saved at $TRIVY_CYCLONEDX_FILE."
else
    echo "Trivy CycloneDX conversion failed!" >&2
    exit 1
fi

# Upload Syft SBOM report to Dependency-Track
echo "Uploading Syft SBOM report to Dependency-Track..."
if curl -X "POST" "$DTRACK_API_URL" \
     -H "Content-Type: multipart/form-data" \
     -H "X-Api-Key: $DTRACK_API_KEY" \
     -F "project=$PROJECT_UUID" \
     -F "bom=@$SBOM_REPORT_FILE"; then
    echo "Syft SBOM uploaded successfully."
else
    echo "Failed to upload Syft SBOM to Dependency-Track!" >&2
    exit 1
fi

# Upload Trivy CycloneDX report to Dependency-Track
echo "Uploading Trivy vulnerability report to Dependency-Track..."
if curl -X "POST" "$DTRACK_API_URL" \
     -H "Content-Type: multipart/form-data" \
     -H "X-Api-Key: $DTRACK_API_KEY" \
     -F "project=$PROJECT_UUID" \
     -F "bom=@$TRIVY_CYCLONEDX_FILE"; then
    echo "Trivy vulnerability report uploaded successfully."
else
    echo "Failed to upload Trivy report to Dependency-Track!" >&2
    exit 1
fi

echo "Security scans completed successfully."
```

---

## **6. Running the Script in Devtron CI**
- Add the script to **Devtron CI/CD pipeline** as a post-image-build stage.
- Ensure all **environment variables** are configured in the pipeline.
- The script will:
  1. Install **Syft** and **Trivy** (if not present).
  2. Run security scans on the built image.
  3. Generate SBOM & vulnerability reports.
  4. Upload reports to **Dependency-Track**.

---

## **7. Validating Reports in Dependency-Track**
After the script execution:
1. **Login to Dependency-Track UI**
2. **Go to "Projects"**
3. **Select the respective project**
4. **View uploaded SBOM & vulnerability reports**  
   - Vulnerability trends
   - Dependency insights
   - Security posture of the image

---

## **8. Tools Used**
| Tool  | Purpose |
|-------|---------|
| [Dependency-Track](https://docs.dependencytrack.org/) | SBOM & Vulnerability Management |
| [Syft](https://github.com/anchore/syft) | SBOM Generation |
| [Trivy](https://github.com/aquasecurity/trivy) | Vulnerability Scanning |
| [Devtron](https://devtron.ai/) | CI/CD Pipeline |

---

## **9. Conclusion**
✔ **Automated security scanning in Devtron CI**  
✔ **Syft & Trivy ensure comprehensive SBOM & vulnerability detection**  
✔ **Dependency-Track enables monitoring & risk assessment**  
✔ **Seamless API integration uploads reports automatically**  

🚀 **Secure your DevOps pipeline with automated SBOM scanning & vulnerability tracking!** 🚀

### Kubernetes

[kubectl Commands and Options](https://jamesdefabia.github.io/docs/user-guide/kubectl/kubectl/)   
[kubectl Cheat Sheet](https://www.mirantis.com/blog/kubernetes-cheat-sheet/)   

[Kubernetes the hard way by Kelsey Hightower](https://github.com/kelseyhightower/kubernetes-the-hard-way)   
  - assembling a Kubernetes cluster part by part

[K8S video series - Google CloudOnAir](https://cloudonair.withgoogle.com/events/kubernetes-with-google-cloud)

[Crossplane](crossplane.io)
  - extends Kubernetes control plane to orchestrate life cycle of public cloud-based services, e.g. database instances, VMs, ML jobs, Big Data vlusters.
  - Kubernetes-styled, declarative, API-driven, yaml config, kubectl CLI, IaC
 
[Jib](https://cloud.google.com/java/getting-started/jib)
  - builds container images without Dockers
  - Maven and Gradle plugin available
  - Distroless iamges: only application and runtime dependencies. No package managers, shells and other parts of Linix distribution.
  - security plus, shorter push/pull time.
 
[Skaffold](https://skaffold.dev/)
  - handles workflow for container and Kubernetes Continuous Deployment
  - automates build, debug, test, push, deploy of artifacts
  - automate `kubectl port-forward` to local machine and log aggregation for feedback

[Helm](https://helm.sh/)
  - Kubernetes package manager
  - uses packaging format called __charts__
  - a chart is a collection of files in a directory, used to describe a related set of Kubernetes resources
  - chart can be versioned

[Kustomize](https://kustomize.io/)
  - tool to customize Kubernetes manifest / object
  - to add, update, remove configuration
  - use kustomization file

<hr>

### Container Security
#### [Kube-bench](https://github.com/aquasecurity/kube-bench)
 - [open-source benchmark testing](https://blog.aquasec.com/announcing-kube-bench-an-open-source-tool-for-running-kubernetes-cis-benchmark-tests)
#### [Trivy](https://aquasecurity.github.io/trivy/v0.31.2/)
 - [SBOM, CVEs, IaCs, secrets, licenses scanner](https://github.com/aquasecurity/trivy)
#### [Tracee](https://github.com/aquasecurity/tracee)  
- [eBPF](https://ebpf.io/what-is-ebpf/) runs codes in Linux kernel
- open-source eBPF tool for runtime security observability
- run as daemon set and pod in every node

<hr>

### Cryptography (python) <br>
https://cryptography.io/en/latest/
<p>Crypto101s
https://www.crypto101.io/
<p>Crypto challenges
https://cryptopals.com/

<hr>  

### DevOps<br>
[DevOps](https://cloud.google.com/devops)  
[Jenkins - open source](https://www.jenkins.io/)  
[Continuous Deployment to GKE with Jenkins - archived pages](https://cloud.google.com/kubernetes-engine/docs/archive/jenkins-on-kubernetes-engine)  
[Spinnaker - open-source multi-cloud continuous deployment](https://spinnaker.io/)  
[Tekton - Continuous Delivery Foundation graduated project](https://tekton.dev/)

<hr>

### DevSecOps
- Policies - for security, compliance
- [Open Policy Agent](https://www.openpolicyagent.org/) - single policy framework for cloud-native environment.
- deploy Gatekeeper on Kubernetes cluster to enforce policy
- OPA can integrate with IDE for policy enforcement at point of coding
- [Deploying OPA on GKE step-by-step](https://medium.com/linkbynet/deploying-opa-on-a-gke-cluster-da4d3d77812c)

<hr>

[Compute options on GCP](https://cloud.google.com/blog/topics/developers-practitioners/where-should-i-run-my-stuff-choosing-google-cloud-compute-option)   
[BigTable vs Big Query](https://cloud.google.com/blog/topics/developers-practitioners/bigtable-vs-bigquery-whats-difference)   

<hr>

### How to install Dockers and Kubernetes locally on Windows 11 <br>
1. Check that hardware virtualisation is enabled in BIOS.
    - Press **F2 key** during Windows boot/restart to enter BIOS.
    - Select **Security** tab in BIOS.
    - Enable **Intel (R) Virtualisation Technology if needed.
    - Save and Exit.
2. Enable Microsoft Hyper-V via Windows operating system.
    - **WIN + R** to open Run window.
    - or, Search for and select **Control Panel**.
      - Select **Windows and Features**.
      - Select **Turn Windows Features on or off**.
      - Tick **Windows Hypervisor Platform** and click **OK**.
      - Install and restart accordingly.

![Tick Windows Hypervisor Platform in Windows Features](https://github.com/TCLee-tech/Google/blob/32e0e06495243f19ebeb173cdef1f5381393bd81/images/Windows%20features%20hypervisor%20selection.jpg)

3. Enable WSL2 (Windows Subsystem for Linux 2).
    - Hyper-V is a Type-1 hypervisor.
    - all Linux containers must run inside a Linux virtual machine. Kubernetes control plane also runs on Linux.
    - WSL2 offers deeper integration with Windows host, and memory reclaim (i.e. it only uses right amount of RAM to run Linux kernel.
    - In PowerShell or Windows Command Prompt with **administrator** access, enter `wsl --install`
    - To list Linux distributions installed and the wsl version each installation is set to, enter `wsl -l -v`
    - To change wsl to version 2, enter `wsl --set-version <name-of-Linux-distribution> 2`, e.g. `wsl --set-version Ubuntu-20.04 2`.
    - Reference: [Install Linux on Windows with WSL](https://learn.microsoft.com/en-us/windows/wsl/install) 

4. Install [Docker Desktop for Windows](https://www.docker.com/products/docker-desktop/)
5. Enable Kubernetes in Docker Desktop
    - Right-click on Docker Desktop icon in taskbar.
    - Select **Settings**.
    - Select **Kubernetes** on left-hand side menu
    - Tick **Enable Kubernetes** then **Apply and Restart**.

![Enable Kubernetes in Docker Desktop](https://github.com/TCLee-tech/Google/blob/29db0d6df000108b6d07f092bc82e63600233534/images/Enable%20Kubernetes%20in%20Docker%20Desktop.jpg)

Reference: [Microsoft Learn - Kubernetes on Windows](https://learn.microsoft.com/en-us/virtualization/windowscontainers/kubernetes/getting-started-kubernetes-windows)


# Google
* notes in folder
<hr>

### Cryptography (python) <br>
https://cryptography.io/en/latest/
<p>Crypto101s
https://www.crypto101.io/
<p>Crypto challenges
https://cryptopals.com/

<hr>  

### DevOps<br>
https://cloud.google.com/devops
<p>Jenkins on Kubernetes
https://cloud.google.com/architecture/jenkins-on-kubernetes-engine
<p>Continuous Deployment to GKE with Jenkins
https://cloud.google.com/architecture/continuous-delivery-jenkins-kubernetes-engine

<hr>

### DevSecOps
- Policies - for security, compliance
- [Open Policy Agent](https://www.openpolicyagent.org/) - single policy framework for cloud-native environment.
- deploy Gatekeeper on Kubernetes cluster to enforce policy
- OPA can integrate with IDE for policy enforcement at point of coding
- [Deploying OPA on GKE step-by-step](https://medium.com/linkbynet/deploying-opa-on-a-gke-cluster-da4d3d77812c)

<hr>

### Terraform<br>
Hasicorp Configuration Language. Cloud agnostic.<br>
Solution for Challenge Lab: Automating Infrastructure on Google Cloud.<br>

<hr>

### Data Science<br>
Solution for Challenge Lab: Insights from Data with Big Query<br>

<hr>

### Machine Learning<br>
Solution for Challenge Lab: Integrate with Machine Learning APIs<br>

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

Reference: [Installing Kubernetes components individually on Windows](https://learn.microsoft.com/en-us/virtualization/windowscontainers/kubernetes/getting-started-kubernetes-windows)


# LAB
## Prerequisites

### Operating System

**Ubuntu 20.04/22.04**

### Docker & Docker Compose

- **Docker & Docker Compose**: Required to orchestrate and deploy the 5G Core and UERANSIM containers.
    
    Update Packages and Install Dependencies
    
    ```bash
    sudo apt-get update
    sudo apt-get install ca-certificates curl gnupg
    ```
    
    Add Docker's Official GPG Key
    
    ```bash
    sudo install -m 0755 -d /etc/apt/keyrings
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
    sudo chmod a+r /etc/apt/keyrings/docker.gpg
    ```
    
    Set Up the Repository
    
    ```bash
    echo \
      "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
      $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
      sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
    ```
    
    Install Docker Engine and Docker Compose
    
    ```bash
    sudo apt-get update
    sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
    ```
    
    [Optional] Docker Group Setup for Convenience
    
    ```bash
    sudo usermod -aG docker $USER
    newgrp docker
    ```
    

### Python

- **Python 3.8+**: Ensure you have Python 3 installed on your machine.
- **Required Libraries**: The script relies on specific packages for data manipulation and plotting. Install them via `pip`:
    
    ```bash
    pip install pandas numpy matplotlib
    ```
    

### gtp5g

- Old version ( free5GC v3.0.6 / v3.2.1 )
    
    ```bash
    git clone --branch v0.4.0 https://github.com/free5gc/gtp5g.git
    ```
    
- New version ( free5GC v4.2.1 )
    
    ```bash
    git clone --branch v0.9.16 https://github.com/free5gc/gtp5g.git
    ```
    

## Getting Started

- Switch to the Correct Branch
    
    Before running any containers, you **must** switch to the Git branch that matches your current OS Kernel and the Core Network version you intend to test.
    
    ```bash
    git checkout <branch_name>
    ```
    
    | **Test** | **Version** | **Supported Kernel** | OS version | **Git Branch** |
    | --- | --- | --- | --- | --- |
    | Old version 
    C | free5GC  v3.2.1
    Open5GS v2.3.6 | 5.4.0 | **Ubuntu 20.04** | Old-Version-CP |
    | Old version | free5GC  v3.0.6
    Open5GS v2.3.6 | 5.4.0 | **Ubuntu 20.04** | Old-Version-UP |
    | New version | free5GC  v4.2.1
    Open5GS v2.7.6 | not limited | **Ubuntu 22.04** | New-Version |
- Build/Get Core Network images
    - free5GC
    
    ```bash
    cd free5gc
    make base
    docker compose pull
    ```
    
    Note: You need to remove old Docker images during testing older versions because the Control Plane and User Plane use different versions
    
    - Open5GS
    
    ```bash
    cd open5gs
    make
    ```
    
- Run Experiment
    - Control Plane
        
        run_control_plane_experiment.sh
        
    - User Plane
        
        run_user_plane_experiment.sh
        

## Acknowledgments

### **References**

- https://github.com/free5gc/free5gc
- https://github.com/aligungr/UERANSIM
- https://github.com/free5gc/gtp5g
- https://github.com/open5gs/open5gs
- https://github.com/PORVIR-5G-Project
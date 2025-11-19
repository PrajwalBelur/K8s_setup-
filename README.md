# K8s_setup-

# kubeadm single-run scripts (Ubuntu)

Below you can find two scripts to prepare Ubuntu nodes for a kubeadm Kubernetes cluster:

- `master.sh` — prepares the master node and initializes the control plane. After running it, run:

- kubeadm token create –print-join-command

- on the master to get the `kubeadm join ...` command for workers.

- `worker.sh` — prepares a worker node and prompts you to paste the `kubeadm join ...` command (the script will execute it with `sudo` and add `--v=5`).

Notes:
- Run as:

- chmod +x master.sh worker.sh
sudo ./master.sh        # on master
sudo ./worker.sh        # on worker (paste join command when prompted)

--------- master.sh ---------
#!/usr/bin/env bash
# master.sh - single-run setup script for Kubernetes master (Ubuntu)
set -euo pipefail

echo "Starting master node setup..."

# 1) kubectl download & install (client)
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl.sha256"
echo "$(cat kubectl.sha256)  kubectl" | sha256sum --check
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
chmod +x kubectl
mkdir -p ~/.local/bin
mv ./kubectl ~/.local/bin/kubectl
# user should ensure ~/.local/bin is in PATH if not already
kubectl version --client

# disable swap
sudo swapoff -a

# Load modules at bootup
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system

## Install CRI-O Runtime
sudo apt-get update -y
sudo apt-get install -y software-properties-common curl apt-transport-https ca-certificates gpg

sudo curl -fsSL https://pkgs.k8s.io/addons:/cri-o:/prerelease:/main/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/cri-o-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/cri-o-apt-keyring.gpg] https://pkgs.k8s.io/addons:/cri-o:/prerelease:/main/deb/ /" | sudo tee /etc/apt/sources.list.d/cri-o.list

sudo apt-get update -y
sudo apt-get install -y cri-o

sudo systemctl daemon-reload
sudo systemctl enable crio --now
sudo systemctl start crio.service

echo "CRI runtime installed successfully"

# Add Kubernetes APT repository and install required packages
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update -y
sudo apt-get install -y kubelet="1.29.0-*" kubectl="1.29.0-*" kubeadm="1.29.0-*"
sudo apt-get update -y
sudo apt-get install -y jq

sudo systemctl enable --now kubelet
sudo systemctl start kubelet

# Master-only steps: initialize control plane
echo "Pulling kubeadm images..."
sudo kubeadm config images pull

echo "Initializing kubeadm control plane..."
sudo kubeadm init

echo "Copying admin kubeconfig to $HOME/.kube/config..."
mkdir -p "$HOME"/.kube
sudo cp -i /etc/kubernetes/admin.conf "$HOME"/.kube/config
sudo chown "$(id -u)":"$(id -g)" "$HOME"/.kube/config

# Install Calico network plugin (as in original script)
echo "Applying Calico network plugin..."
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.0/manifests/calico.yaml

echo "Master setup finished."
echo
echo "To generate join command for workers, run on master:"
echo "  kubeadm token create --print-join-command"
echo
echo "Remember to allow port 6443 from worker nodes in your security group/firewall."



------- worker.sh ----------

#!/usr/bin/env bash
# worker.sh - single-run setup script for Kubernetes worker (Ubuntu)
# This script prepares a worker node and then asks you to paste the kubeadm join command.
set -euo pipefail

echo "Starting worker node setup..."

# 1) kubectl download & install (client)
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl.sha256"
echo "$(cat kubectl.sha256)  kubectl" | sha256sum --check
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
chmod +x kubectl
mkdir -p ~/.local/bin
mv ./kubectl ~/.local/bin/kubectl
# user should ensure ~/.local/bin is in PATH if not already
kubectl version --client

# disable swap
sudo swapoff -a

# Load modules at bootup
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system

## Install CRI-O Runtime
sudo apt-get update -y
sudo apt-get install -y software-properties-common curl apt-transport-https ca-certificates gpg

sudo curl -fsSL https://pkgs.k8s.io/addons:/cri-o:/prerelease:/main/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/cri-o-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/cri-o-apt-keyring.gpg] https://pkgs.k8s.io/addons:/cri-o:/prerelease:/main/deb/ /" | sudo tee /etc/apt/sources.list.d/cri-o.list

sudo apt-get update -y
sudo apt-get install -y cri-o

sudo systemctl daemon-reload
sudo systemctl enable crio --now
sudo systemctl start crio.service

echo "CRI runtime installed successfully"

# Add Kubernetes APT repository and install required packages
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update -y
sudo apt-get install -y kubelet="1.29.0-*" kubectl="1.29.0-*" kubeadm="1.29.0-*"
sudo apt-get update -y
sudo apt-get install -y jq

sudo systemctl enable --now kubelet
sudo systemctl start kubelet

# Worker-only steps: reset pre-flight checks and then ask for join
echo "Resetting kubeadm pre-flight (if previous state exists)..."
sudo kubeadm reset pre-flight

# Prompt for join command (user will paste the full 'kubeadm join ...' command)
echo
echo "Paste the full kubeadm join command you obtained from the master (example starts with 'kubeadm join')"
echo "It will be executed with sudo. If you don't have it yet, generate it on master using:"
echo "  kubeadm token create --print-join-command"
echo
read -rp "Paste kubeadm join command (or press ENTER to skip): " JOIN_CMD

if [ -n "$JOIN_CMD" ]; then
  echo "Running join command..."
  # Run with sudo to ensure proper privileges
  sudo bash -c "$JOIN_CMD --v=5"
  echo "Join command executed."
else
  echo "No join command provided. Exiting. Paste the join command and run it manually when ready."
fi

echo "Worker setup finished. Verify from master with: kubectl get nodes"

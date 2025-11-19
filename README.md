# K8s_setup-

# kubeadm single-run scripts (Ubuntu)

This repository contains two scripts to prepare Ubuntu nodes for a kubeadm Kubernetes cluster:

- `master.sh` — prepares the master node and initializes the control plane. After running it, run:

- kubeadm token create –print-join-command

- on the master to get the `kubeadm join ...` command for workers.

- `worker.sh` — prepares a worker node and prompts you to paste the `kubeadm join ...` command (the script will execute it with `sudo` and add `--v=5`).

Notes:
- These scripts are written for Ubuntu (Xenial or later) and use CRI-O as the container runtime as in the original instructions.
- They do **not** store or generate any join tokens in the repo. You must generate the join command on the master and paste it into the worker script at runtime.
- Run as:

- chmod +x master.sh worker.sh
sudo ./master.sh        # on master
sudo ./worker.sh        # on worker (paste join command when prompted)

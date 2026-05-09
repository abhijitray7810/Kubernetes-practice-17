# Kubernetes Practice Repository 🚀
 
This repository contains hands-on examples and configurations for working with Kubernetes. It is designed to help beginners and intermediate users understand core Kubernetes concepts through practical YAML files and command references.

---

## 📂 Repository Structure

Each folder in this repository represents a specific Kubernetes resource or concept:

* **All-command/** – Collection of commonly used Kubernetes commands
* **Apache-Namespace/** – Namespace configuration for Apache workloads
* **Config/** – ConfigMaps and configuration files
* **Cron-Job/** – Scheduled jobs using CronJob resources
* **Cron-jobs-2/** – Additional CronJob examples
* **Daemonset/** – DaemonSet configurations
* **Deployments/** – Deployment manifests and examples
* **Jobs/** – Kubernetes Job configurations
* **Nginx-Dm/** – Nginx deployment steps
* **Nginx-Namespace/** – Namespace setup for Nginx
* **Nginx-Pods/** – Pod configurations for Nginx
* **Nginx-ReplicaSets/** – ReplicaSet examples
* **PersistentVolume/** – Persistent Volume (PV) definitions
* **PersistentVolum_claim/** – Persistent Volume Claim (PVC) configs
* **Pods/** – Basic Pod configurations
* **Service/** – Service definitions for exposing applications

---

## 🧠 What You Will Learn

* Kubernetes architecture basics
* Creating and managing Pods
* Deployments and ReplicaSets
* Services and networking
* Persistent storage (PV & PVC)
* Jobs and CronJobs
* Namespaces and resource isolation
* ConfigMaps and configuration handling

---

## ⚙️ Prerequisites

Before using this repository, make sure you have:

* Kubernetes cluster (Minikube / Kind / Cloud cluster)
* `kubectl` CLI installed and configured
* Basic understanding of containerization (Docker recommended)

---

## 🚀 How to Use

1. Clone the repository:

   ```bash
   git clone https://github.com/abhijitray7810/Kubernetes.git
   cd Kubernetes
   ```

2. Apply any configuration:

   ```bash
   kubectl apply -f <file-name>.yaml
   ```

3. Verify resources:

   ```bash
   kubectl get all
   ```

---

## 📌 Example

Deploy a Pod:

```bash
kubectl apply -f Pods/Pod.yaml
```

Check status:

```bash
kubectl get pods
```

---

## 📖 Notes

* Each folder may contain its own README or command reference
* YAML files are structured for learning and experimentation
* Modify configurations as needed for your environment

---

## 🤝 Contributing

Feel free to fork this repository and contribute by adding more examples or improving existing ones.

---

## 📜 License

This project is open-source and available under the MIT License.

---

## ⭐ Support

If you find this repository helpful, consider giving it a star!

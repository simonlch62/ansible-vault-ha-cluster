# Ansible - Cluster HashiCorp Vault HA

Deploiement automatise d'un cluster Vault 3 noeuds avec stockage Raft integre et load balancer HAProxy.

## 1. Presentation du projet

Ce projet Ansible deploie :

- **3 noeuds Vault** en haute disponibilite (Raft integre)
- **1 HAProxy** comme point d'entree unique avec health check automatique

En cas de panne d'un noeud, Vault elit un nouveau leader et HAProxy bascule le trafic automatiquement.

| Machine | Role | IP |
|---|---|---|
| vault-01 | Vault (Raft leader/follower) | 192.168.92.141 |
| vault-02 | Vault (Raft leader/follower) | 192.168.92.130 |
| vault-03 | Vault (Raft leader/follower) | 192.168.92.131 |
| haproxy-01 | Load balancer | 192.168.92.129 |

<img width="1146" height="791" alt="image" src="https://github.com/user-attachments/assets/a16a7f3f-a52a-4886-bb27-abf582e4a173" />


## 2. Structure du projet

```
├── ansible.cfg
├── inventory/
│   └── hosts.yml
├── group_vars/
│   └── vault_servers.yml
├── playbooks/
│   ├── site.yml
│   └── haproxy.yml
└── roles/
    ├── vault/
    │   ├── defaults/main.yml
    │   ├── tasks/main.yml
    │   ├── handlers/main.yml
    │   └── templates/
    │       ├── vault.hcl.j2
    │       └── vault.service.j2
    └── haproxy/
        ├── defaults/main.yml
        ├── tasks/main.yml
        ├── handlers/main.yml
        └── templates/
            └── haproxy.cfg.j2
```

## 3. Procedure d'exploitation

### 3.1 Pre-requis client (machine de controle)

- Linux
- Ansible installe :
  ```bash
  sudo dnf install ansible-core
  ```
- Collection ansible.posix (pour firewalld) :
  ```bash
  ansible-galaxy collection install ansible.posix
  ```
- Cle SSH Ed25519 :
  ```bash
  ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519
  ```

### 3.2 Ressources minimales

| Machine | OS | vCPU | RAM | Disque |
|---|---|---|---|---|
| vault-01 | RHEL 9 | 2 | 2 Go | 20 Go |
| vault-02 | RHEL 9 | 2 | 2 Go | 20 Go |
| vault-03 | RHEL 9 | 2 | 2 Go | 20 Go |
| haproxy-01 | RHEL 9 | 1 | 512 Mo | 10 Go |

### 3.3 Pre-requis serveur (4 machines RHEL 9)

Sur chaque serveur, en root :

```bash
useradd -m -s /bin/bash deploy
echo "deploy:test" | chpasswd # ne pas oublier de le supprimer
echo "deploy ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/deploy
chmod 440 /etc/sudoers.d/deploy
```

Puis depuis le client, copier la cle SSH :

```bash
for IP in 192.168.92.141 192.168.92.130 192.168.92.131 192.168.92.129; do
  ssh-copy-id -i ~/.ssh/id_ed25519.pub deploy@$IP
done
```

### 3.4 Deploiement

```bash
ansible-playbook playbooks/site.yml --diff
```

### 3.5 Initialisation du cluster Vault

A faire une seule fois apres le premier deploiement.

**Sur vault-01 :**

```bash
export VAULT_ADDR='http://127.0.0.1:8200'
vault operator init
```

Sauvegarder les 5 cles Unseal et le Root Token.

**Unsealer les 3 noeuds** (vault-01, puis vault-02, puis vault-03) :

```bash
export VAULT_ADDR='http://127.0.0.1:8200'
vault operator unseal <cle_1>
vault operator unseal <cle_2>
vault operator unseal <cle_3>
```

**Verifier le cluster :**

```bash
vault login <root_token>
vault operator raft list-peers
```

Resultat attendu :

```
Node        Address                State       Voter
----        -------                -----       -----
vault-01    192.168.92.141:8201    leader      true
vault-02    192.168.92.130:8201    follower    true
vault-03    192.168.92.131:8201    follower    true
```

### 3.6 Verification HAProxy

```bash
curl http://192.168.92.129:8200/v1/sys/health
```

Page de stats : `http://192.168.92.129:8404/stats`

### 3.7 Acces a Vault

- **UI Web** : `http://192.168.92.129:8200/ui`
- **API / CLI** : `export VAULT_ADDR='http://192.168.92.129:8200'`
- **GitLab CI** : variable `VAULT_ADDR = http://192.168.92.129:8200`

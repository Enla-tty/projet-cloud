# Projet Cloud - WordPress sur Azure

Déploiement automatisé d'une infrastructure WordPress sur Azure avec Ansible.

## Architecture

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   Load Balancer │────▶│   app-vm-1      │────▶│  MySQL Flexible │
│   (IP publique) │     │   WordPress     │     │  Server         │
└─────────────────┘     └─────────────────┘     └─────────────────┘
         │              ┌─────────────────┐              ▲
         └─────────────▶│   app-vm-2      │              │
                        │   WordPress     │──────────────┘
                        └─────────────────┘
                                 ▲
                                 │
                        ┌─────────────────┐
                        │   jumpoff-vm    │
                        │   (Bastion SSH) │
                        └─────────────────┘
```

## Structure du projet

```
projet-cloud/
├── site.yml                    # Déploiement avec cloud-init
├── site-full.yml               # Déploiement complet Ansible SSH
├── deploy-wordpress.yml        # Déploiement WordPress via SSH
├── playbooks/
│   ├── 01-network.yml                    # Création du réseau
│   ├── 02-mysql.yml                      # Création du serveur MySQL
│   ├── 03-infrastructure.yml             # VMs avec cloud-init (site.yml)
│   └── 03-infrastructure-no-cloudinit.yml # VMs sans cloud-init (site-full.yml)
├── files/
│   └── cloud-init-app.yml.j2   # Template cloud-init pour les VMs
├── vars/
│   ├── secrets.yml             # Credentials (chiffré avec ansible-vault)
│   └── secrets.yml.example     # Exemple de secrets
├── README.md                   # Documentation
└── requirements.txt            # Dépendances Python
```

## Prérequis

```bash
# Activer le virtualenv
source venv/bin/activate

# Connexion Azure
az login

# Préparer les secrets
cp vars/secrets.yml.example vars/secrets.yml
nano vars/secrets.yml
ansible-vault encrypt vars/secrets.yml
```

## Déploiement

### Option 1 : Déploiement complet (recommandé)

Déploie l'infrastructure ET WordPress via Ansible SSH :

```bash
ansible-playbook site-full.yml --ask-vault-pass
```

### Option 2 : Déploiement avec cloud-init

Déploie l'infrastructure, WordPress est installé automatiquement via cloud-init :

```bash
ansible-playbook site.yml --ask-vault-pass
```

## Connexion aux VMs

```bash
# IP du jumpoff
JUMPOFF_IP=$(az vm show -g wordpress-rg -n jumpoff-vm -d --query publicIps -o tsv)

# Connexion au jumpoff
ssh -i ~/.ssh/id_rsa azureuser@$JUMPOFF_IP

# Connexion directe aux VMs applicatives (via jumpoff)
ssh -i ~/.ssh/id_rsa -J azureuser@$JUMPOFF_IP azureuser@10.0.0.20
ssh -i ~/.ssh/id_rsa -J azureuser@$JUMPOFF_IP azureuser@10.0.0.21
```

## Vérification

```bash
# Vérifier MySQL
az mysql flexible-server show \
  -g wordpress-rg \
  -n wordpress-mysql-flex \
  --query "{name:name, state:state, fqdn:fullyQualifiedDomainName}" \
  -o table

# Tester WordPress (via FQDN)
curl -I http://wordpressdemohexagone.denmarkeast.cloudapp.azure.com
```

## Nettoyage

```bash
# Supprimer toutes les ressources
az group delete -n wordpress-rg --yes --no-wait
```

## Différence entre les méthodes

| | `site.yml` | `site-full.yml` |
|---|---|---|
| **Méthode** | Cloud-init | Ansible SSH |
| **WordPress** | Installé automatiquement au boot | Installé par Ansible après création VMs |
| **Contrôle** | Limité (script cloud-init) | Total (playbook Ansible) |
| **Utilisation** | Déploiement rapide | Déploiement contrôlé |

## Détails techniques

- **OS**: Ubuntu 22.04 LTS
- **PHP**: 8.1
- **Web Server**: Nginx
- **Database**: Azure MySQL Flexible Server 8.0
- **Load Balancer**: Azure Standard LB (HTTP/HTTPS)
- **Réseau**: VNet 10.0.0.0/24 avec 3 subnets

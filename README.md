# 1. Créer le venv et se mettre dedans
python3 -m venv .venv
source .venv/bin/activate

# 2. Installer Ansible et ansible-galaxy
pip install ansible
ansible-galaxy collection install azure.azcollection --force

# 3. Installer les requirements et voir la liste
pip install -r ~/.ansible/collections/ansible_collections/azure/azcollection/requirements.txt
ansible-galaxy collection list


# 3. Se connecter à Azure
az login

# 4. Lancer le playbook
ansible-playbook infrastructure_wordpress.yml

# 5. connexion aux vm
on vérifie l'existence clef ssh :
ls -la ~/.ssh/id_rsa ~/.ssh/id_rsa.pub

on vérifie les droits :
chmod 600 ~/.ssh/id_rsa
chmod 644 ~/.ssh/id_rsa.pub

IP publique :
az vm show -g wordpress-rg -n jumpoff-vm -d --query publicIps -o tsv

IP privée :
az vm show -g wordpress-rg -n jumpoff-vm -d --query privateIps -o tsv
az vm show -g wordpress-rg -n app-vm-1 -d --query privateIps -o tsv
az vm show -g wordpress-rg -n app-vm-2 -d --query privateIps -o tsv

Depuis notre pc pour copier la clef ssh sur jumpoff:
scp -i ~/.ssh/id_rsa ~/.ssh/id_rsa azureuser@20.123.9.94:~/.ssh/id_rsa

connexion à la vm jumpoff :
ssh -i ~/.ssh/id_rsa azureuser@20.123.9.94
perm de la clef : chmod 600 ~/.ssh/id_rsa

OU sans copie de la clef ssh:

Depuis notre pc pour se connecter à app-vm-1
ssh -i ~/.ssh/id_rsa -J azureuser@$JUMPOFF_IP azureuser@10.0.0.20

Depuis notre pc pour se connecter à app-vm-1
ssh -i ~/.ssh/id_rsa -J azureuser@$JUMPOFF_IP azureuser@10.0.0.21

Depuis jumpoff :
se connecter à app-vm-1 : ssh -i ~/.ssh/id_rsa azureuser@10.0.0.20
se connecter à app-vm-2 : ssh -i ~/.ssh/id_rsa azureuser@10.0.0.21





# 1. Exporter la subscription
export AZURE_SUBSCRIPTION_ID=$(az account show --query id -o tsv)

# 2. Préparer les secrets
cp vars/secrets.yml.example vars/secrets.yml
nano vars/secrets.yml
ansible-vault encrypt vars/secrets.yml

# 3. Lancer
ansible-playbook site.yml --ask-vault-pass

az provider register --namespace Microsoft.DBforMySQL
az provider show --namespace Microsoft.DBforMySQL --query registrationState -o tsv
attendre Registred puis re : ansible-playbook site.yml --ask-vault-pass

# 4. Vérifier MySQL
az mysql flexible-server show \
  -g wordpress-rg \
  -n wordpress-mysql-flex \
  --query "{name:name, state:state, fqdn:fullyQualifiedDomainName}" \
  -o table

# 5. Tester depuis le jumpoff
JUMPOFF_IP=$(az vm show -g wordpress-rg -n jumpoff-vm -d --query publicIps -o tsv)
ssh -i ~/.ssh/id_rsa azureuser@$JUMPOFF_IP

# Depuis le jumpoff :
mysql -h wordpress-mysql-flex.wordpress.mysql.database.azure.com \
      -u wpadmin -p'W0rdPr3ss@2024Secure!' \
      -e "SHOW DATABASES;"
# 1. Installer la collection Azure pour Ansible
ansible-galaxy collection install azure.azcollection

# 2. Installer les dépendances Python
pip install -r ~/.ansible/collections/ansible_collections/azure/azcollection/requirements.txt

# 3. Se connecter à Azure
az login

# 4. Lancer le playbook
ansible-playbook deploy_infra.yml

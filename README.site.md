# Только статический IP
ansible-playbook site_static_IP.yml \
--vault-password-file ~/.ansible_vault_pass

# Полный деплой всего стенда
ansible-playbook site_full_install.yml \
-i inventories/dev/hosts.yml 
--vault-password-file ~/.ansible_vault_pass
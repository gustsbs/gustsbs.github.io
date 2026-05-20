# 🌐Guia de Comandos PROXMOX

### Ativa o modo manutenção de um nó
O Proxmox evacuará automaticamente as VMs/CTs sob regras de Alta Disponibilidade (HA)


```bash
ha-manager crm-command node-maintenance enable pve1
 ```

### Define a flag 'noout' no Ceph
Impede que o Ceph tente replicar os OSDs do nó que sairá do ar, evitando sobrecarga na rede


 ```bash
 ceph osd set noout
 ```

# Monitora em tempo real a migração e o estado do HA
```bash
watch -n 2 ha-manager status
```
# Lista as VMs rodando localmente no nó saudável para confirmar a aterrissagem
```bash
qm list
```

# Desativa o modo de manutenção no pve1 (reintegrando o nó ao HA do cluster)

```bash
ha-manager crm-command node-maintenance disable pve1
```
# Remove a trava do Ceph para que o storage volte a sincronizar os OSDs do pve1
```bash
ceph osd unset noout
```




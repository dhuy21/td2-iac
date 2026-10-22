# TD2 — Industrialiser K3s avec Terraform, Ansible et Helm

Pipeline IaC qui déploie l'application 3-tiers **NovaShop** (nginx, API Python,
PostgreSQL) sur une VM K3s mono-nœud dans OpenStack (DevStack, projet `asl-project`).

## Rôle de chaque outil

| Outil | Rôle | Dossier |
|---|---|---|
| Terraform | Provisionnement : SG, VM, IP flottante (le socle ASL est lu via data sources) | `terraform/` |
| Ansible | Configuration : prérequis système, installation de K3s, récupération du kubeconfig | `ansible/` |
| Helm | Déploiement applicatif : 3 charts indépendants (`postgres`, `backend`, `frontend`) | `helm/` |

## Rejouer le pipeline

```bash
source ~/devstack/asl-openrc                      # identifiants ASL uniquement
cd terraform && cp example.tfvars terraform.tfvars  # renseigner le login
terraform init && terraform apply
terraform output -raw ansible_inventory > ../ansible/inventory.ini

cd ../ansible && ansible-playbook playbook.yml    # K3s + kubeconfig (API sur 127.0.0.1)
./tunnel.sh &                                     # tunnel SSH vers l'API K3s (6443)

export KUBECONFIG=$PWD/kubeconfig
kubectl create namespace td2
cd ../helm
helm install postgres ./postgres -n td2 --wait
helm install backend  ./backend  -n td2 --wait
helm install frontend ./frontend -n td2 --wait
```

Application : `http://<floating_ip>`.

Destruction : `helm uninstall` des 3 releases, `kubectl -n td2 delete pvc --all`
(le PVC du StatefulSet n'est pas supprimé par Helm), puis `terraform destroy`.

## Bonus — compteur de visites

Le dépôt contient la version avec l'endpoint `GET /api/visits` et la card « Visites totales ».
La table `visits` est créée par `init.sql` lors d'un déploiement neuf ; sur une base
existante, elle doit être ajoutée par migration (le script d'init ne rejoue pas).

## Choix et écarts par rapport à l'énoncé

- **cloud-init** : `chpasswd.list` plutôt qu'une redéfinition de l'utilisateur `ubuntu`,
  qui empêchait l'injection de la keypair Nova.
- **Rôle kubeconfig** : l'adresse de l'API est une variable (`kubeconfig_api_host`,
  fixée à `127.0.0.1` dans le playbook). Le kubeconfig est directement utilisable
  avec le tunnel SSH et le playbook reste idempotent (`changed=0`).
- **Annotations `checksum/*`** dans les Deployments backend et frontend : une
  modification de ConfigMap déclenche un rollout lors d'un `helm upgrade`.
- **`.gitignore`** : `ansible/kubeconfig` au lieu de `kubeconfig`, qui excluait aussi
  le dossier du rôle `roles/kubeconfig/`.

## Sécurité (labo)

- Port 6443 fermé dans le SG : l'API K3s n'est accessible que par tunnel SSH.
- Exclus du dépôt : `terraform.tfstate`, `*.tfvars`, `inventory.ini`, `ansible/kubeconfig`.
- Les mots de passe en clair (`azerty`, Postgres `changeme`) sont assumés dans ce labo
  isolé ; ils seront protégés au TD3 (Ansible Vault, Sealed Secrets).

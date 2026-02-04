🚀 Workshop Partie 2 : Structure Pro & La Maison SVG
Pré-requis : Avoir terminé la Partie 1.

🎯 Objectif
Maintenant que nous savons déployer une page simple, nous allons :

Restructurer le projet pour séparer proprement le code (Packer, Ansible, Source).

Remplacer la page d'accueil par un projet complexe (Dessin SVG).

Mettre à jour le déploiement sans casser le cluster (Rolling Update).

📂 1. Réorganisation (Mode Industriel)
Pour ne pas mélanger les fichiers à la racine, nous allons adopter l'arborescence demandée par le professeur.

Exécutez cette commande dans le terminal pour créer les dossiers et déplacer les fichiers :

Bash
# Création des dossiers
mkdir -p packer/www ansible

# Nettoyage des anciens fichiers temporaires (si existants)
rm -f index.html nginx.pkr.hcl playbook.yml inventory.ini
🏠 2. Le Code Source (La Maison)
Au lieu d'écrire du HTML à la main, nous allons télécharger le code source de la "Maison SVG".

Bash
# Télécharger le fichier index.html directement dans le dossier source
curl -o packer/www/index.html https://raw.githubusercontent.com/bstocker/Maison_SVG/main/index.html
🏗️ 3. Mise à jour de Packer
Nous devons dire à Packer de chercher le fichier dans le nouveau dossier packer/www/.

Créez (ou écrasez) le fichier packer/nginx.pkr.hcl :

Fichier : packer/nginx.pkr.hcl

Terraform
packer {
  required_plugins {
    docker = {
      version = ">= 1.0.8"
      source  = "github.com/hashicorp/docker"
    }
  }
}

variable "tag" {
  type    = string
  default = "2.0.0"  # On passe en version 2 !
}

source "docker" "nginx_custom" {
  image  = "nginx:latest"
  commit = true
}

build {
  name = "mon-nginx-custom"
  sources = ["source.docker.nginx_custom"]

  # Copie le fichier depuis le nouveau dossier www
  provisioner "file" {
    source      = "www/index.html"
    destination = "/usr/share/nginx/html/index.html"
  }

  post-processor "docker-tag" {
    repository = "mon-nginx-custom"
    tags       = [var.tag]
  }
}
Action : Construire la v2

Bash
cd packer
packer build nginx.pkr.hcl
cd ..
🌉 4. Import dans le Cluster
N'oubliez jamais cette étape ! Sinon Kubernetes ne verra pas la nouvelle version.

Bash
k3d image import mon-nginx-custom:2.0.0 -c lab
🚀 5. Mise à jour d'Ansible
Nous allons mettre à jour nos scripts Ansible pour utiliser la nouvelle image et la nouvelle structure.

1. L'inventaire (ansible/hosts.ini)

Ini, TOML
[local]
localhost ansible_connection=local
2. Le Playbook (ansible/deploy.yml) Nous mettons à jour l'image vers la version 2.0.0.

YAML
- name: Mettre à jour vers la Maison SVG
  hosts: localhost
  gather_facts: false
  vars:
    image_version: "2.0.0"  # Variable pour changer facilement de version

  tasks:
    - name: S'assurer que le namespace existe
      kubernetes.core.k8s:
        name: web-service
        api_version: v1
        kind: Namespace
        state: present

    - name: Mettre à jour le Déploiement
      kubernetes.core.k8s:
        definition:
          apiVersion: apps/v1
          kind: Deployment
          metadata:
            name: nginx-deployment
            namespace: web-service
          spec:
            replicas: 1
            selector:
              matchLabels:
                app: my-nginx
            template:
              metadata:
                labels:
                  app: my-nginx
              spec:
                containers:
                - name: nginx
                  image: "mon-nginx-custom:{{ image_version }}" # Utilisation de la variable
                  imagePullPolicy: Never
                  ports:
                  - containerPort: 80

    - name: S'assurer que le Service est là
      kubernetes.core.k8s:
        definition:
          apiVersion: v1
          kind: Service
          metadata:
            name: nginx-service
            namespace: web-service
          spec:
            selector:
              app: my-nginx
            ports:
              - protocol: TCP
                port: 80
                targetPort: 80
            type: LoadBalancer
Action : Lancer la mise à jour

Bash
cd ansible
ansible-playbook -i hosts.ini deploy.yml
cd ..
🌍 6. Vérification Finale
Si tout s'est bien passé, Kubernetes a remplacé l'ancien conteneur "Hello World" par la "Maison SVG".

Action 1 : Force le redémarrage (Optionnel mais recommandé) Pour être sûr que le pod a bien pris la nouvelle image tout de suite :

Bash
kubectl delete pod --all -n web-service
Action 2 : Ouvrir l'accès Nous utilisons un port différent (9090) pour éviter les problèmes de cache du navigateur.

Bash
kubectl port-forward svc/nginx-service -n web-service 9090:80
Action 3 : Voir le résultat

Allez dans l'onglet PORTS.

Cliquez sur le globe à côté du port 9090.

Admirer la maison ! 🏠

Félicitations ! Vous maîtrisez maintenant la mise à jour d'application via Infrastructure as Code.
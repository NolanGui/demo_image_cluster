🚀 Workshop : From Image to Cluster
Auteur : Nolan Sujet : Automatisation du cycle de vie applicatif (Packer → Ansible → Kubernetes)

🎯 Objectif de l'atelier
L'objectif est de comprendre comment passer du code source à une application qui tourne en production sans aucune intervention manuelle "fragile".

Nous allons :

Construire une image Docker personnalisée (Nginx) avec Packer.

Importer cette image dans un cluster K3d.

Déployer l'application automatiquement avec Ansible.

🛠️ 1. Pré-requis (Installation)
Dans votre environnement GitHub Codespaces, les outils nécessaires ne sont pas tous installés par défaut.

Copiez et exécutez ce bloc de commande pour préparer votre environnement :

Bash

# Installer l'outil K3d
curl -s https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash

# Créer le cluster "lab" (ça prend environ 1 minute)
k3d cluster create lab \
  --servers 1 \
  --agents 2

# 1. Installer Packer (Binaire direct pour éviter les erreurs de dépôt)
wget https://releases.hashicorp.com/packer/1.10.0/packer_1.10.0_linux_amd64.zip
sudo apt update && sudo apt install unzip -y
unzip packer_1.10.0_linux_amd64.zip
sudo mv packer /usr/local/bin/
rm packer_1.10.0_linux_amd64.zip

# 2. Installer Ansible et les modules Kubernetes
pip install ansible kubernetes
ansible-galaxy collection install kubernetes.core

# 3. Vérifier les installations
packer --version
ansible --version
Assurez-vous également que votre cluster K3d est démarré (séquence 2 du sujet original).

📦 2. Préparation de l'application (Source)
Créez un fichier index.html à la racine. C'est le cœur de notre site web.

Fichier : index.html

HTML
<!DOCTYPE html>
<html>
<head>
    <title>Projet DevOps</title>
    <style>
        body { font-family: sans-serif; text-align: center; padding-top: 50px; background-color: #f0f0f0; }
        h1 { color: #333; }
    </style>
</head>
<body>
    <h1>Bienvenue sur le Cluster de [Votre Prénom] !</h1>
    <p>Déployé automatiquement avec Packer et Ansible.</p>
</body>
</html>
🏗️ 3. Construction de l'image (Packer)
Nous allons utiliser Packer pour créer une "Golden Image" contenant notre site.

Créez le fichier de configuration Packer.

Fichier : nginx.pkr.hcl

Terraform
packer {
  required_plugins {
    docker = {
      version = ">= 1.0.8"
      source  = "github.com/hashicorp/docker"
    }
  }
}

source "docker" "nginx_custom" {
  image  = "nginx:latest"
  commit = true
}

build {
  name = "mon-nginx-custom"
  sources = ["source.docker.nginx_custom"]

  # Copie le fichier local vers l'image
  provisioner "file" {
    source      = "index.html"
    destination = "/usr/share/nginx/html/index.html"
  }

  # Tag l'image pour la retrouver facilement
  post-processor "docker-tag" {
    repository = "mon-nginx-custom"
    tags       = ["v1"]
  }
}
Action : Lancer la construction

Bash
packer init .
packer build nginx.pkr.hcl
✅ Résultat attendu : Packer confirme la création de l'image mon-nginx-custom:v1.

🌉 4. Le Pont vers K3d (L'étape piège)
K3d tourne dans des conteneurs isolés. Il ne voit pas les images que vous venez de créer localement. Il faut les importer manuellement.

Action : Importer l'image

Bash
k3d image import mon-nginx-custom:v1 -c lab
🚀 5. Déploiement (Ansible)
Au lieu de taper des commandes kubectl manuelles, nous allons décrire l'état désiré de notre infrastructure.

Créez d'abord l'inventaire pour dire à Ansible d'agir en local.

Fichier : inventory.ini

Ini, TOML
[local]
localhost ansible_connection=local
Puis créez le fichier deploy pour le déploiement.

Fichier : deploy.yml

YAML
- name: Déployer Nginx Custom sur K3d
  hosts: localhost
  gather_facts: false
  tasks:
    - name: Créer le namespace dédié
      kubernetes.core.k8s:
        name: web-service
        api_version: v1
        kind: Namespace
        state: present

    - name: Déployer l'application
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
                  image: mon-nginx-custom:v1
                  imagePullPolicy: Never
                  ports:
                  - containerPort: 80

    - name: Exposer le service
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
Action : Lancer le déploiement

Bash
ansible-playbook -i hosts.ini playbook.yml
✅ Résultat attendu : Ansible affiche des tâches en "changed" (jaune/vert).

🌍 6. Vérification et Accès
L'application tourne dans le cluster. Pour y accéder depuis votre navigateur, nous allons créer un tunnel.

Action 1 : Vérifier le statut

Bash
kubectl get pods -n web-service
(Vous devez voir un statut Running)

Action 2 : Ouvrir l'accès (Port Forward)

Bash
# On redirige le port 8080 local vers le port 80 du service
kubectl port-forward svc/nginx-service -n web-service 8080:80
Action 3 : Voir le résultat

Allez dans l'onglet PORTS de votre Codespace.

Repérez le port 8080.

Cliquez sur l'icône 🌐 (Open in Browser).

🎉 Bravo ! Votre application personnalisée est en ligne.

🔄 Exercice pour aller plus loin
Modifiez le fichier index.html (changez une couleur ou le texte).

Rejouez la chaîne complète :

packer build ...

k3d image import ...

kubectl delete pod --all -n web-service (Pour forcer le redémarrage immédiat)

Rafraîchissez votre page web pour voir le changement.
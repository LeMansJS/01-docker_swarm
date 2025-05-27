# 🎤 Introduction à Docker Swarm

Bienvenue sur le dépôt de la présentation sur : **"Introduction à Docker Swarm"**.

Cette conférence s'adresse aux développeurs, étudiants et curieux souhaitant comprendre les bases de l'orchestration de conteneurs avec Docker Swarm, un outil natif et simple pour automatiser le déploiement d'applications conteneurisées.

![Présentation Docker Swarm](image.png)

## 🎥 Replay

📺 Vous avez manqué la conférence en direct ? Pas de souci !  
Le replay est disponible sur YouTube :  
👉 [Le replay]([https://youtube.com](https://youtu.be/ZozLdz9b0dY?si=EeywCiIHl8goI-lV))

## 🛠️ Étape 00 - Mise à jour du VPS et installation des packages nécessaire

```bash
sudo apt update && sudo apt upgrade -y

sudo apt install curl git unzip -y
```

### Installer un firewall

```bash
sudo apt install ufw -y
sudo ufw allow OpenSSH
sudo ufw allow 2376/tcp #Docker API port
sudo ufw allow 2377/tcp #Docker Swarm port
sudo ufw allow 7946/tcp #Docker Swarm intra-node communication
sudo ufw allow 7946/udp
sudo ufw allow 4789/udp #Overlay network traffic
sudo ufw allow 80/tcp #HTTP traffic
sudo ufw allow 443/tcp #HTTPS traffic
sudo ufw enable
```

### Installer un fail2ban

```bash
sudo apt install fail2ban -y
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
```


## 🛠️ Étape 01 - Installer Docker

```bash
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
```

```bash
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

```bash
sudo systemctl start docker
sudo systemctl enable docker
```

```bash
sudo docker run hello-world
```

## 🛠️ Étape 02 - Initialisé sont cluster Docker Swarm

```bash
docker swarm init --advertise-addr <IPV4 DE TON SERVEUR MANAGER>
```

## 🛠️ Étape 03 - Faire rejoindre le cluster sur les autres serveurs

```bash
docker swarm join --token <TOKEN DE TON SERVEUR MANAGER> <IPV4 DE TON SERVEUR MANAGER>:2377
```

Pour vérifié que tous tes serveurs ont bien rejoint je cluster fais cette commande :

```bash
docker node ls
```

Si tu vois tous tes serveur en "Ready" et "Active", tu as réussi

## 🛠️ Étape 04 (optionnel) - Configurer les DNS pour docker

```bash
sudo nano /etc/docker/daemon.json
```

```json
{
    "dns": ["8.8.8.8", "8.8.4.4"]
}
```

```bash
sudo systemctl restart docker
```

## 🛠️ Étape 05 - Configuration de Caddy Docker Proxy

```bash
docker network create -d overlay caddy_network
```

```bash
mkdir -p /srv/docker/caddy
touch /srv/docker/caddy/compose.yml
```

```bash
nano /srv/docker/caddy/compose.yml
```

```yml
services:
  caddy:
    image: lucaslorentz/caddy-docker-proxy:latest
    ports:
      - 80:80
      - 443:443
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - caddy_data:/data
      - caddy_config:/config
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.role == manager
    networks:
      - caddy_network # Caddy est connecté à ce réseau

networks:
  caddy_network:
    external: true # Réseau externe partagé

volumes:
  caddy_data:
  caddy_config:
```

```bash
docker stack deploy -c /srv/docker/caddy/compose.yml caddy
```

Pour vérifier que c'est bon :

```bash
docker service ls
```

## 🛠️ Étape 06 - Pointer son domaine vers le serveur manager

Ajouter un enregistrement de type A de votre domaine qui pointe vers l'IPV4 de votre serveur manager

## 🛠️ Étape 07 - Déployer son premier service. (whoami)

```bash
mkdir -p /srv/docker/whoami
touch /srv/docker/whoami/compose.yml
```

```bash
nano /srv/docker/whoami/compose.yml
```

```yml
services:
  whoami:
    image: traefik/whoami
    deploy:
      replicas: 3 # Utilisation de 3 répliques pour tester le load balancing
    labels:
      - caddy=<TON NOM DE DOMAINE>
      - caddy.reverse_proxy="{{upstreams 80}}" # Redirection des requêtes vers l'application whoami sur le port 80
    networks:
      - caddy_network # Connexion au réseau partagé

networks:
  caddy_network:
    external: true # Assure toi que ce réseau est déjà créer
```

```bash
docker stack deploy -c /srv/docker/whoami/compose.yml whoami
```

Pour vérifié que c'est bon :

```bash
docker service ls
```

## 🛠️ Étape 08 - Scaler son service
Une fois ton service est déployé, tu peux changer dynamiquement le nombre de répliques sans devoir redéployer le stack complet.

### 🔼 Scaler

Par exemple, pour passer à 5 répliques :

```bash
docker service scale whoami_whoami=5
```

Ou bien :

```bash
docker service update --replicas=5 whoami_whoami
```

Ou bien encore en modifiant le nombre de réplicas dans le fichier Compose, puis en redéployant la stack.

### 🔄 Vérification du scaling

Pour vérifier que le scaling a bien été appliqué :

```bash
docker service ps whoami_whoami
```

Tu devrais voir le bon nombre de tâches actives réparties sur les différents nœuds.

## 🛠️ Étape 09 - Mettre à jour et rollback un service

Docker Swarm permet de faire des mises à jour progressives (rolling updates) de tes services sans coupure.
<br>En cas de problème, tu peux effectuer un rollback automatique à la version précédente.

### 🔄 Mettre à jour l'image du service

Par exemple, pour passer à une version plus récente (même image ou autre image compatible) :

```bash
docker service update --image traefik/whoami:latest whoami_whoami
```

> ℹ️ Cela mettra à jour les 3 réplicas un par un, sans interrompre le service.

### 🔙 Faire un rollback (retour arrière)

Si une mise à jour échoue ou pose problème, tu peux revenir à l’état précédent avec :

```bash
docker service rollback whoami_whoami
```

### 🔍 Vérifier l’état de la mise à jour

Pendant ou après la mise à jour, tu peux vérifier l’état de tes conteneurs :

```bash
docker service ps whoami_whoami
```

Tu verras l’historique des tâches (anciens conteneurs arrêtés, nouveaux en cours).

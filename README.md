# TP Docker 

### DataBase

[Lien vers le dossier db](../postgres)

[Lien vers le dossier http](../http)

[Lien vers le dossier backend](../backend)

- FROM : Définit l'image de base pour la construction de la nouvelle image Docker.
- ENV : Définit des variables d'environnement dans le conteneur Docker en cours de construction.
- COPY : Copie des fichiers ou des répertoires depuis le système de fichiers local dans l'image 

**Qu 1-1**

*Commande de base de Docker :*
```Dockerfile
docker build -t jbrider/frontdev .
```
On ajoute "-t" pour étiqueter l'image construite
```Dockerfile
 docker run -p 8080:8080 --name backend --network app-network jbrider/backend
```
On utilise le localhost, d'où le port 8080.

On oublie pas de copier les fichiers vers un répertoire spécifique à l'intérieur du conteneur Docker en cours de construction : 
```Dockerfile
COPY 01-CreateScheme.sql /docker-entrypoint-initdb.d
COPY 02-InsertData.sql /docker-entrypoint-initdb.d
```

### Backend API

**Qu 1-2**

Grâce au multistage build, l'image finale ne contient que le fichier JAR et les dépendances d'exécution nécessaires. L'image Docker finale est donc plus petite et plus optimisée. La première étape, contenant les dépendances de construction et les fichiers intermédiaires est jetée après la construction.

### Htttp server et Link application

**Qu 1-3**

*Commandes docker-compose :*
```Dockerfile
docker compose up
```
Cette commande est utilisée pour démarrer tous les services dans le docker-compose.yml.

**Qu 1-4**

*Détail de mon docker compose :*

Pour chacun des services : 
- *build* indique le chemin (path)
- *networks* indique le network configurer plus tôt
- *ports* configuration des ports, seulement pour le proxy
- *depends_on* lorsque Docker compose démarre ou redémarre le service, il s'assurera également que l'autre service, préciser ici, soit également redémarrer. Auquel cas il attendra

### Publish

**Qu 1-5**

```Dockerfile
# On se log
docker login
# On met un tag
docker tag my-database USERNAME/my-database:1.0
# On push pour publier
docker push USERNAME/my-database:1.0  
```


# TP Git Action

[Lien vers le dossier github](./.github/workflows)

**Qu 2-1**

Un "conteneur de test" dans GitHub Actions est un environnement isolé basé sur Docker où les tests de votre projet sont exécutés, assurant ainsi la cohérence, la reproductibilité et la facilité de configuration des tests.

**Qu 2-2**

```yml
name: CI devops 2023  # Nom du workflow

on:
  push:  # Événement déclencheur : push sur les branches main et develop
    branches:
      - main
      - develop
  pull_request:  # Événement déclencheur : pull request

jobs:
  test-backend:  # Job pour tester le backend
    runs-on: ubuntu-22.04  # Environnement d'exécution : Ubuntu 22.04
    steps:
      - name: Checkout code  # Étape : Récupérer le code source
        uses: actions/checkout@v2.5.0

      - name: Set up JDK 17  # Étape : Configuration de JDK 17
        uses: actions/setup-java@v3
        with:
          java-version: '17'
          distribution: "adopt"

      - name: Log in to Docker Hub  # Étape : Connexion à Docker Hub
        uses: docker/login-action@v1
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}  # Nom d'utilisateur secret
          password: ${{ secrets.DOCKERHUB_TOKEN }}  # Jeton secret

      - name: Build and test with Maven  # Étape : Construction et test avec Maven
        run: | 
              cd simple-api 
              mvn clean test
```

**Qu 2-3**

On va simplement modifier les dernières lignes afin de faire des tests avec sonar.

```Dockerfile
- name: Build and test with Maven
        run: mvn -B verify sonar:sonar -Dsonar.projectKey=gjbox -Dsonar.organization=gjbox -Dsonar.host.url=https://sonarcloud.io -Dsonar.login=${{ secrets.SONAR_TOKEN }}  --file ./backend/simple-api-student/pom.xml
```

La variable "SONAR_TOKEN" est au préalable configurée comme variable d'environnement secrète dans git.

# TP Ansible

[Lien vers le playbook](./ansible/playbook.yml)

**Qu 3-1**

*Setup.yml*
```Dockerfile
all:
 vars:
   ansible_user: "{{ ansUSER }}"
   ansible_ssh_private_key_file: /id_rsa
 children:
   prod:
     hosts: "{{ ansHOST }}" 
```

Les variables d'environnement sont définies [ici](./ansible/group_vars/all.yml).
Le fichier est cependant crypter grâce à vault.

```Dockerfile
ansible all -i inventories/setup.yml -m ping
```

Cette commande exécute une tâche Ansible utilisant le module "ping" sur tous les hôtes répertoriés dans le fichier inventories/setup.yml pour vérifier leur accessibilité et leur fonctionnement.

**Qu 3-2**

*Note :* pour être parfaitement honnête j'ai utilisé l'image Docker de Théo, car à ce moment là j'ai rencontré une erreur pour publish sur dockerhub et je voulais avancer le TP. Veuillez m'excuser et ne pas trop m'en tenir rigueur.

Cordialement,

*Le playbook [ici](./ansible/playbook.yml)*
```Dockerfile
- hosts: all
  gather_facts: false
  become: true

  roles:
    - docker
    - create_network
    - launch_database
    - launch_app
    - launch_proxy
    - launch_front
```

*Docker*
```Dockerfile
- name: Install device-mapper-persistent-data
  yum:
    name: device-mapper-persistent-data
    state: latest

- name: Install lvm2
  yum:
    name: lvm2
    state: latest

- name: add repo docker
  command:
    cmd: sudo yum-config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo

- name: Install Docker
  yum:
    name: docker-ce
    state: present

- name: Install python3
  yum:
    name: python3
    state: present

- name: Install docker with Python 3
  pip:
    name: docker
    executable: pip3
  vars:
    ansible_python_interpreter: /usr/bin/python3

- name: Make sure Docker is running
  service: name=docker state=started
  tags: docker
```

*Create Network*
```Dockerfile
- name: Create a network
  community.docker.docker_network:
    name: "{{ networkName }}"

```

*Launch app*
```Dockerfile
- name: Run App Container
  docker_container:
    name: "{{ contAPP }}"
    image: "theoahga/my-backend:latest"
    networks:
      - name: "{{ networkName }}"
    env:
      DBURL: "{{ dbURL }}"
      DBUSER: "{{ dbUSER }}"
      DBPWD: "{{ dbPWD }}"
```

*Launch Database*
```Dockerfile
- name: Run Database Container
  docker_container:
    name: "{{ contDB }}"
    image: "theoahga/my-database:latest"
    networks:
      - name: "{{ networkName }}"

```

*Launch Proxy*
```Dockerfile
- name: Run Proxy Container
  docker_container:
    name: "{{ contPROXY }}"
    image: "theoahga/my-http:latest"
    networks:
      - name: "{{ networkName }}"
    ports: 
     - 8080:80
```

*Merci de m'avoir lu !*

**Bonne soirée**
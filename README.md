
# **Applicatie met Docker Compose**

![archi](architectuur.png)

## **1. Containers Online Zetten met Docker Compose**

### **a. Installeer Docker en de Docker Compose Plugin**

Zorg ervoor dat Docker en de Docker Compose plugin op je machine zijn geïnstalleerd.  
Volg de officiële handleiding hier:  
https://docs.docker.com/engine/install/

### **b. Start de Containers**

1. Start alle containers:  

Start de frontend, backend, database, Jenkins en Traefik met:

```bash
docker compose up -d
```

Voor specifieke services (bv. Jenkins):

```bash
cd jenkins
docker compose up -d
```

2. **Controleer de status van de containers**:

```bash
docker ps
```

## **2. Aanpassingen in de Compose Files voor Eigen Situatie**

### **a. `docker-compose.yml` (frontend/backend/mysql)**

- **Domeinnamen**:  
    Pas de labels aan om jouw domein te gebruiken:

```yaml
- "traefik.http.routers.frontend.rule=Host(`jouwdomein`)"
```

- Enviroment variables:
	maak een .env bestand en vul je eigen database-gegevens in:
	
```yaml
MYSQL_HOST=mysql
MYSQL_USER=user
MYSQL_ROOT_PASSWORD=root_password
MYSQL_DATABASE=database
MYSQL_PASSWORD=password
```

### **b. `traefik-compose.yml` en `traefik.yml`**

- **Domeinnamen**:  
    Vervang de domeinnamen door jouw eigen domeinen.  
    Labels in `traefik-compose.yml`:

```yaml
- "traefik.http.routers.traefik.rule=Host(`traefik.jouwdomein`)"
```

- **E-mail voor Let's Encrypt**:  
	Pas je e-mailadres aan voor certificaten in `traefik.yml`:

```yaml
email: jouw-email@example.com
```

- **DNS Configuratie**:  
	Zorg dat jouw domeinen verwijzen naar het IP-adres van je server via je DNS-provider.

### **c. Jenkins Compose File**

- Pas de domeinnaam voor Jenkins aan:

```yaml
- "traefik.http.routers.jenkins.rule=Host(`jenkins.jouwdomein`)"
```

# **Setup van Jenkins**

### **a. Jenkins Starten en Toegang Krijgen**

1. Open Jenkins in de browser via:

```
https://jenkins.jouwdomein
```

2. **Initial Admin Password**:  
	Zoek het initiële wachtwoord in de Jenkins-container:

```bash
docker logs jenkins
```

3. Volg de installatie wizard om Jenkins te configureren.

### **b. Credentials toevoegen aan jenkins**

1. maak een ssh keypair aan:

```bash
ssh-keygen
```

2. Installeer java op je vm:

```bash
sudo apt install openjdk-17-jre
```

3. maak jenkins user aan op je vm:

```bash
useradd -m -s /bin/bash jenkins
```

4. Zet de public key van je eerder aangemaakte keypair in de authorized_kets van deze jenkins user.

5. ga op jenkins naar manage jenkins -> Credentials -> global -> Add credential:

- ssh username with private key
- ID: kies je zelf
- username: jenkins
- private key van jenkins user

6. Voeg ook alle enviroment variabales als secret text toe bij credentials.

### **c. agent toevoegen aan Jenkins**

1. ga op jenkins naar manage jenkins -> Nodes -> New node:

- permanent yes
- 1 executor
- remote root is `/home/jenkins`
- labels zijn je domeinnamen met spaties ertussen
- launch method –> SSH
- Host: domijnnaam of IP van je server
- Credentials: de credentials die je eerder maakte
- Host key verification strategy: none

### **d. Pipeline Script Configureren**

1. Ga naar **Jenkins → New Item** en maak een nieuw Pipeline project aan.

3. Voeg het volgende **Pipeline script** toe:

```groovy
pipeline {
    agent { label 'jouw domein' }

    stages {
        stage('Checkout') {
            steps {
                git credentialsId: 'gekozen id van credentials', branch: 'main', url: 'jouw repo'
            }
        }

        stage('Build and deploy') {
            environment{
                MYSQL_HOST = credentials('mysql_host')
                MYSQL_USER = credentials('mysql_user')
                MYSQL_ROOT_PASSWORD = credentials('mysql_root_password')
                MYSQL_DATABASE = credentials('mysql_database')
                MYSQL_PASSWORD = credentials('mysql_password')
            }
            steps{
                sh "docker compose up --build -d"
            }
        }
    }
}
```


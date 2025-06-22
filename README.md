# BobApp

Clone project:

> git clone XXXXX

## Front-end 

Go inside folder the front folder:

> cd front

Install dependencies:

> npm install

Launch Front-end:

> npm run start;

### Docker

Build the container:

> docker build -t bobapp-front .  

Start the container:

> docker run -p 8080:8080 --name bobapp-front -d bobapp-front

## Back-end

Go inside folder the back folder:

> cd back

Install dependencies:

> mvn clean install

Launch Back-end:

>  mvn spring-boot:run

Launch the tests:

> mvn clean install

### Docker

Build the container:

> docker build -t bobapp-back .  

Start the container:

> docker run -p 8080:8080 --name bobapp-back -d bobapp-back 

## Vue d’ensemble & Explication de la CI/CD

Ce document décrit la chaîne **CI/CD** mise en place pour le projet **Bobapp** : un back‑end **Spring Boot** et un front‑end **Angular**.  L’objectif est de garantir que chaque mise à jour du code du projet passe les tests, respecte les standards qualité et se déploie automatiquement sur **Docker Hub**.

---

## 1 · Étapes du workflow GitHub Actions

La chaîne CI/CD repose sur trois workflows distincts définis dans `.github/workflows/` :

### 🔹 test-coverage.yml

Exécution des **tests** et génération de la **couverture de code** pour le front‑end et le back‑end.

| Étape                        | Objectif                                                                              | Exécuté sur   | Déclencheurs        | Commandes |
| ---------------------------- | ------------------------------------------------------------------------------------- | ------------- | ------------------- | -------- |
| **Checkout & Cache**         | Récupérer le code et restaurer les caches Maven/NPM                                   | ubuntu-latest | `push` `pull request`           | `- uses: actions/checkout@v4` |
| **Frontend · Build & Tests** | Installer les dépendances, lancer les tests Jasmine, générer le rapport de couverture | Node 14       |              | `- npm install` `- npm run test-coverage` |
| **Publish Coverage Report**  | Archiver les rapports XML                                                             | GitHub Pages  |  | `- uses : actions/upload-artifact@v4` |

### 🔹 sonar-analysis.yml

Analyse statique du code via **SonarCloud** pour les deux parties (front et back).

| Étape               | Objectif                                                                           | Exécuté sur      | Déclencheurs         | Commandes |
| ------------------- | ---------------------------------------------------------------------------------- | ---------------- | -------------------- | ---------- |
| **Checkout**        | Récupérer le code                                                                  | ubuntu-latest    | action tests validée | `- uses: actions/checkout@v4` |
| **Download artifact**| Pour la partie front-end, télécharger le rapport depuis l'archive                 | ubuntu-latest    |  | `- uses: actions/download-artifact@v4` |
| **SonarCloud Scan** (front-end) | Lancer `sonar-scanner` avec les chemins vers les sources et rapports de couverture | SonarScanner CLI |                      | `- uses: SonarSource/sonarqube-scan-action@v5` |
| **SonarCloud Scan** (backend-end) | Lancer les tests puis lancer `sonar-scanner` avec les chemins vers les sources et rapports de couverture | SonarScanner CLI |                      | `- run: mvn -B verify org.sonarsource.scanner.maven:sonar-maven-plugin:sonar` |

Le scan est paramétré pour reconnaître les deux modules (frontend et backend) et publier les résultats vers **SonarCloud**. La qualité du code est contrôlée via une **Quality Gate**.

### 🔹 docker-publish.yml

Build et publication des **images Docker** front‑end et back‑end sur **Docker Hub**.

| Étape                       | Objectif                                                                      | Exécuté sur   | Déclencheurs                 | Commandes |
| --------------------------- | ----------------------------------------------------------------------------- | ------------- | ---------------------------- | ---------- |
| **Checkout & Setup**        | Récupérer le code, configurer Docker Buildx et les permissions de publication | ubuntu-latest | action sonar analyse validée | `- uses: actions/checkout@v4` |
| **Login to DockerHub**      | Authentifier le workflow sur Docker Hub via secrets                           | Docker CLI    |                              | `- uses: docker/login-action@v3` |
| **Backend · Docker Build**  | Construire l'image `bobapp-back:latest`                                       | Docker Buildx |                              | `- run: docker build -t ${{ secrets.DOCKERHUB_USERNAME }}/bobapp-front:latest .` |
| **Frontend · Docker Build** | Construire l'image `bobapp-front:latest`                                      | Docker Buildx |                              | `- run: docker build -t ${{ secrets.DOCKERHUB_USERNAME }}/bobapp-back:latest .` |
| **Push Backend Image**      | Pousser les images back‑end vers le registre Docker                           | Docker CLI    |                              | `- run: docker push ${{ secrets.DOCKERHUB_USERNAME }}/bobapp-front:latest` |
| **Push Frontend Image**     | Pousser les images front‑end vers le registre Docker                          | Docker CLI    |                              | `- run: docker push ${{ secrets.DOCKERHUB_USERNAME }}/bobapp-back:latest` |

Chaque workflow est chaîné logiquement :

- Si les tests échouent, l’analyse Sonar n’est pas déclenchée
- Si la Quality Gate Sonar échoue, les images Docker ne sont pas publiées

---

## 2 · KPIs & Quality Gate proposées

| KPI                                     | Seuil requis | Justification                                      |
| --------------------------------------- | ------------ | -------------------------------------------------- |
| **Code coverage front-end et back-end** | ≥ 80 %       | Limiter les régressions et garantir un socle testé |
| **New Blocker Issues (Sonar)**          | 0            | Aucune nouvelle erreur critique n’est tolérée      |
| **Duplications Density**                | < 3 %        | Réduire la dette technique                         |
| **Build & Test Duration**               | ≤ 5 min      | Expérience développeur et feedback rapide          |

Ces seuils sont configurés dans la **Quality Gate** Sonar.

---

## 3 · Premières métriques (exécution nº1)

| Mesure               | Valeur actuelle | Statut par rapport au seuil |
| -------------------- | --------------- | --------------------------- |
| Couverture back‑end  | **38.8%**         | ❌ à améliorer              |
| Couverture front‑end | **47.6%**      | ❌ à améliorer              |
| New Blocker Issues   | **0**           | ✅ conforme                  |
| Duplications         | **0%**          | ✅ conforme                 |
| Durée pipeline       | **3 min 22s**    | ✅ conforme                  |

Les rapports détaillés sont générés et publiés dans l’onglet **Actions → Artifacts** ainsi que sur **SonarCloud**.

---

## 4 · Analyse des “Notes et avis” utilisateurs

| Gravité    | Commentaire utilisateur                                                                                                                                   | Problème identifié                                                 | Action prioritaire                                                           |
| ---------- | --------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------ | ---------------------------------------------------------------------------- |
| 🔴 Haute   | “Je mets une étoile car je ne peux pas en mettre zéro ! Impossible de poster une suggestion de blague, le bouton tourne et fait planter mon navigateur !” | Crash lors de la soumission du formulaire « Suggestion de blague » | Reproduire, écrire test E2E, corriger le front‑end (gestion promise/timeout) |
| 🔴 Haute   | “#BobApp j’ai remonté un bug sur le post de vidéo il y a deux semaines et il est encore présent !!!”                                                      | Bug non résolu sur l’upload ou l’embed de vidéo                    | Ajouter test de régression, investiguer service vidéo                        |
| 🟠 Moyenne | “Ça fait une semaine que je ne reçois plus rien, j’ai envoyé un email il y a 5 jours mais toujours pas de nouvelles…”                                     | Rupture du flux de notifications / support client lent             | Monitorer file de messages & SLA support                                     |
| 🟠 Moyenne | “J’ai supprimé ce site de mes favoris ce matin, dommage.”                                                                                                 | Perte de fidélité due aux bugs récurrents                          | Plan de fiabilité + communication changelog                                  |

Prioriser les deux premiers tickets ; ils bloquent des fonctionnalités clés et génèrent le plus de frustration.

---

---

## 6 · Secrets & configuration

- **SONAR\_TOKEN\_FRONTEND/SONAR\_TOKEN\_BACKEND**: jeton SonarCloud
- **DOCKER\_HUB\_USERNAME / DOCKER\_HUB\_TOKEN**

Ces secrets sont stockés dans *Settings → Secrets & variables → Actions*.

---

## 7 · Liens utiles

- Repository GitHub : [https://github.com/EryanGauvrit/oc-master-p10](https://github.com/EryanGauvrit/oc-master-p10)
- *Workflows CI/CD* : `.github/workflows/test-coverage.yml`, `sonar-analysis.yml`, `docker-publish.yml`
- Tableau SonarCloud : [https://sonarcloud.io/organizations/eryangauvrit/projects](https://sonarcloud.io/organizations/eryangauvrit/projects)
- Images docker hub :
  - Front-end : [https://hub.docker.com/repository/docker/eryangauvrit/bobapp-front/general](https://hub.docker.com/repository/docker/eryangauvrit/bobapp-front/general)
  - Back-end : [https://hub.docker.com/repository/docker/eryangauvrit/bobapp-back/general](https://hub.docker.com/repository/docker/eryangauvrit/bobapp-back/general)
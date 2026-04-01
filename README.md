# TP Final - Integration Logicielle

## Membres du groupe
- KOURAOGO Wendbark Joel Faïsal 1er (Jumeau)
- COULIBALY Abdoul Rachid
- PARE Kontama Léandre Bénilde

## 1) Presentation du projet
Ce projet deploie une stack complete d'integration logicielle avec Docker Compose :
- MariaDB (base GLPI)
- GLPI (ITSM)
- Prometheus (collecte metriques)
- cAdvisor (metriques conteneurs)
- Grafana (visualisation)

Objectif : lancer tout l'environnement en une commande, puis analyser la BDD GLPI et visualiser metiers + monitoring dans Grafana.

Note de compatibilite: les images GLPI courantes demandent MariaDB/MySQL a l'installation. Cette implementation utilise donc MariaDB pour garantir un deploiement fonctionnel sur macOS ARM.

## 2) Prerequis
- Docker Engine >= 24.x
- Docker Compose v2 >= 2.20
- RAM recommandee : 4 Go minimum (8 Go conseille)
- OS testes : Linux/macOS (sur macOS, certains montages cAdvisor peuvent varier)

## 3) Ligne directrice (strategie de realisation)
1. Monter d'abord l'infrastructure Docker Compose.
2. Verifier MariaDB -> GLPI -> Prometheus/cAdvisor -> Grafana dans cet ordre.
3. Analyser la base GLPI avec SQL (fichier `analyse/analyse_bdd_glpi.md`).
4. Construire le dashboard metier GLPI (MySQL datasource).
5. Construire le dashboard monitoring (Prometheus datasource).
6. Documenter les resultats, commandes et retours d'experience dans ce README.

## 4) Etapes de demarrage
1. Cloner le depot puis se placer dans le dossier du projet.
2. Copier les variables d'environnement :
   ```bash
   cp .env.example .env
   ```
3. Demarrer toute la stack :
   ```bash
   docker compose up -d
   ```
4. Suivre les logs GLPI (premier boot peut prendre 1-2 minutes) :
   ```bash
   docker compose logs -f glpi
   ```
5. Verifier les services :
   ```bash
   docker compose ps
   ```

## 5) URLs d'acces
- GLPI : http://localhost:8080
- Grafana : http://localhost:3000
- Prometheus : http://localhost:9090
- cAdvisor : http://localhost:8081
- cAdvisor metriques : http://localhost:8081/metrics
- Prometheus targets : http://localhost:9090/targets

## 6) Identifiants par defaut
- Grafana:
  - utilisateur : `admin`
  - mot de passe : `admin`
- GLPI : utiliser les identifiants initialises a l'installation GLPI (selon image/version).

## 7) Verification Prometheus (Partie 4)
### Targets configures
- `prometheus` -> `localhost:9090`
- `cadvisor` -> `cadvisor:8080`
- `glpi` -> `glpi:80`

Statut attendu : `UP` pour `prometheus` et `cadvisor` (et `glpi` si endpoint disponible).

### Difference entre `scrape_interval` et `evaluation_interval`
- `scrape_interval` : frequence de collecte des metriques depuis les targets.
- `evaluation_interval` : frequence d'evaluation des regles Prometheus (alerting/recording rules).

### Requete PromQL
```promql
rate(container_cpu_usage_seconds_total{name!=""}[5m])
```
Interpretation : taux d'augmentation de la consommation CPU (en secondes CPU/seconde) par conteneur nomme, calcule sur une fenetre glissante de 5 minutes. Cette valeur peut etre multipliee par 100 pour un affichage en pourcentage.

## 8) Analyse BDD GLPI
Voir le livrable : `analyse/analyse_bdd_glpi.md`.

## 9) Arret et nettoyage
- Arret simple :
  ```bash
  docker compose down
  ```
- Arret + suppression volumes (nettoyage complet) :
  ```bash
  docker compose down -v
  ```

## 10) Difficultes rencontrees et solutions
- Le premier demarrage de GLPI est plus long que les autres services, ce qui peut donner l'impression d'un blocage. Solution : suivre `docker compose logs -f glpi`.
- La connexion Grafana -> MariaDB echoue si les variables d'environnement ne sont pas alignees entre `.env` et provisioning datasource. Solution : centraliser les secrets dans `.env`.
- Les dates GLPI sont souvent stockees sous forme de timestamp Unix. Solution : utiliser `FROM_UNIXTIME()` dans les requetes SQL Grafana.
- Les metriques cAdvisor exposent aussi des conteneurs systeme. Solution : filtrer avec `name!=""` dans les requetes PromQL.
- Sur certains environnements (surtout macOS), les chemins montes pour cAdvisor peuvent differer. Solution : adapter les volumes montes en lecture seule selon l'hote.

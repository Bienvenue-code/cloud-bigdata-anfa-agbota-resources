\# Rendu - Séance 2

\*\*Nom et prénom :\*\* AGBOTA Adjo Anne Bienvenue Sika
\*\*Identifiant GitHub :\*\* Bienvenue-code
\*\*Date de soumission :\*\* 25/06/2026

\## Résumé de la séance

Cette séance a porté sur Docker en profondeur, à travers une situation-problème concrète : un data engineer qui doit empaqueter un script PySpark avec toutes ses dépendances pour qu'il s'exécute de façon identique partout, sans réinstallation manuelle.

Le cours a d'abord distingué la virtualisation classique (machines virtuelles, isolées par un hyperviseur, avec un système d'exploitation complet par instance) de la conteneurisation (processus isolés partageant le même noyau Linux). 

Les conteneurs démarrent en quelques millisecondes, ont une empreinte de quelques mégaoctets et permettent une densité de centaines à milliers d'instances sur une même machine, contre quelques dizaines pour des VM. Trois types d'hyperviseurs ont été présentés : 
. type 1 (bare-metal, comme ESXi ou KVM), 
. type 2 (hébergé, comme VirtualBox), et
. type 3, une zone grise illustrée par Firecracker, utilisé notamment par AWS Lambda.

L'histoire de la conteneurisation a été retracée depuis le chroot (1979), en passant par les FreeBSD Jails (2000) et les Solaris Zones (2004), jusqu'à LXC en 2008, qui constitue la fondation technique sur laquelle Docker s'est appuyé. Docker, lancé en mars 2013, n'a pas inventé la conteneurisation mais l'a démocratisée grâce à une CLI simple, un format d'image portable et Docker Hub. L'Open Container Initiative a été fondée en 2015 pour standardiser le format des images et des runtimes, donnant la pile actuelle : runc → containerd → Docker Engine → BuildKit.

Sur le plan technique, un conteneur repose sur trois piliers du noyau Linux : les namespaces (sept types : PID, réseau, montage, UTS, IPC, utilisateur, cgroup), qui isolent ce qu'un processus peut voir ; les cgroups, qui limitent les ressources consommées (CPU, mémoire, I/O) ; et les systèmes de fichiers en union comme OverlayFS, qui superposent des couches de fichiers et permettent le mécanisme de cache de build. Sous macOS et Windows, Docker fonctionne via une machine virtuelle Linux légère, invisible pour l'utilisateur (WSL2 étant recommandé sous Windows).

Le cours a également couvert le vocabulaire fondamental (image, conteneur, Dockerfile, registre, couche, tag) et les instructions clés d'un Dockerfile (FROM, WORKDIR, COPY, RUN, ENV, ARG, EXPOSE, USER, CMD, ENTRYPOINT), ainsi que les bonnes pratiques associées : utiliser une image de base légère, ordonner les instructions pour maximiser la réutilisation du cache, éviter de tourner en root, ne jamais mettre de secrets dans une variable ENV, épingler les versions, et utiliser un .dockerignore. Le multi-stage build a été présenté comme une technique permettant de séparer une étape de construction lourde d'une étape d'exécution légère, en ne copiant dans l'image finale que le résultat nécessaire.

Enfin, Docker Compose a été introduit comme l'outil permettant de décrire un environnement multi-services dans un fichier YAML et de le lancer en une seule commande, avec un réseau partagé créé automatiquement (résolution DNS par nom de service), des volumes nommés pour la persistance, des bind mounts pour le développement, et la possibilité d'attendre qu'un service soit réellement prêt grâce à depends\_on combiné à un healthcheck. Docker Compose a été présenté comme une première étape vers Kubernetes, qui orchestre des conteneurs à travers un cluster de machines plutôt que sur un seul hôte.

\## Étapes principales

1. Écriture du script `analyse\_referentiel.py` (analyse PySpark en mode local du référentiel de bus Anfa) et du `Dockerfile` associé, construction de l'image `anfa-analyse:v1` (taille observée : 1,17 Go).
2. Exécution du conteneur avec montage du référentiel CSV en lecture seule, confirmant le bon fonctionnement de l'analyse (12 lignes de bus, 60 arrêts uniques, 100 bus dont 93 actifs, capacité totale de 4538 places, tarifs moyens par type calculés correctement).
3. Mise en place du `.dockerignore` et observation du mécanisme de cache de Docker.
4. Écriture du `docker-compose.yml` orchestrant trois services : MinIO (stockage objet), Jupyter (notebook d'exploration) et `anfa-app` (image custom construite localement).
5. Recréation du bucket `anfa-raw` et de l'utilisateur applicatif MinIO via le client `mc` (exécuté dans un conteneur Docker), puis rechargement des CSV du référentiel avec le script de la séance 1.
6. Création du notebook `exploration\_minio.ipynb` lisant les données du bucket `anfa-raw` via boto3 et les exploitant avec pandas.

\## Résultats observés

La première construction de l'image a pris environ 199 secondes, principalement consommées par l'installation de Java (31s) et l'installation de PySpark (110s), pour un poids final de 1,17 Go — une taille volontairement disproportionnée pour un script qui ne fait que lire quatre fichiers CSV, et qui illustre bien le coût d'embarquer un runtime JVM complet dans une image Python.

Une fois le `.dockerignore` en place, reconstruire l'image sans aucune modification a pris 2,6 secondes, avec toutes les étapes marquées CACHED. En modifiant uniquement le script Python (ajout d'une ligne d'impression), seule l'étape `COPY . .` a été refaite (0,2 seconde), tandis que l'installation de PySpark est restée en cache : ceci confirme que placer `COPY requirements.txt` et `RUN pip install` avant `COPY . .` protège l'étape la plus coûteuse du cache, même lorsque le code applicatif change fréquemment.

Le lancement de la stack à trois services via `docker compose up -d --build` a fonctionné après résolution d'un conflit de nom de conteneur. La vérification avec `docker compose ps -a` a montré MinIO et Jupyter à l'état "Up (healthy)", et `anfa-app` à l'état "Exited (0)", confirmant que le script batch s'est exécuté puis arrêté normalement sans erreur.

Le notebook Jupyter a confirmé la connexion à MinIO par le nom de service `minio` (résolution DNS interne au réseau Compose, distincte de `localhost` qui désignerait le conteneur Jupyter lui-même), la présence du bucket `anfa-raw`, la liste des quatre fichiers CSV dans `referentiel/`, la lecture du fichier des lignes sous forme de DataFrame pandas (12 lignes de bus avec leurs terminus, nombre d'arrêts et distances), et le calcul du top 3 des lignes les plus longues (Agoè Assiyéyé - Grand Marché : 16,28 km ; Baguida - Adawlato : 13,13 km ; Avédji - Adawlato : 11,67 km), cohérent avec les résultats obtenus côté PySpark.

\## Captures d'écran

\### docker compose ps

!\[docker compose ps](captures/docker-ps.png)

\### Notebook Jupyter

!\[Notebook Jupyter](captures/jupyter-pandas.png)

\## Bonus multi-stage (optionnel)

Non réalisé.

\## Difficultés rencontrées

1. Un conflit de nom de conteneur (`anfa-minio` déjà utilisé par un conteneur existant) a bloqué le premier lancement de la stack ; résolu en supprimant l'ancien conteneur avec `docker rm -f anfa-minio`. 
2. Une erreur `IndentationError` est apparue dans `analyse\_referentiel.py` après l'ajout d'une ligne de test pour observer le cache Docker, la nouvelle ligne n'étant pas correctement indentée par rapport au corps de la fonction ; corrigée en réalignant l'indentation. 
3. Le bucket `anfa-raw` était vide après la reconstruction de l'environnement de la séance 2, le client `mc` n'étant pas installé localement sous Windows ; la création du bucket et de l'utilisateur applicatif a donc été effectuée en exécutant l'image `minio/mc` dans un conteneur connecté au réseau `seance-02\_default`, avant de relancer le script `upload\_referentiel.py` de la séance 1 pour recharger les CSV.

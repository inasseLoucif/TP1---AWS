# TP 1 – Infrastructure AWS – Load Balancer et Instances EC2
Trigramme : ILF
##  Objectif du TP
L’objectif de ce TP est de mettre en place une petite infrastructure AWS composée d’un Application Load Balancer, d’un Target Group et de plusieurs instances EC2 jouant le rôle de serveurs web.
 Chaque instance doit exposer un serveur Apache qui affiche son propre instance-id via les métadonnées EC2, et le trafic doit être distribué par le Load Balancer de manière sécurisée grâce à des Security Groups adaptés.
 
### Partie 1 – Création du Load Balancer et du Target Group
Dans un premier temps, un Target Group nommé ILF-TargetGroup a été créé dans la région eu-west-1 (Europe / Ireland).
 Le type sélectionné est Instances, utilisant le protocole HTTP sur le port 80, avec des health checks HTTP configurés sur le même port afin de vérifier la disponibilité des cibles.
Un Application Load Balancer ILF-LoadBalancer a ensuite été créé.
 Il est associé à des subnets publics de la VPC par défaut et à un Security Group dédié ILF-SecurityGroup-LB.
 Ce Security Group autorise le trafic entrant HTTP (port 80) uniquement depuis l’adresse IP publique utilisée dans les locaux d’Ynov (et éventuellement mon IP personnelle pour certains tests), ce qui limite l’exposition du service.
Le Load Balancer utilise le Target Group ILF-TargetGroup comme destination par défaut de son listener HTTP en port 80.
 Lorsqu’on appelle le DNS public du Load Balancer en HTTP, la requête est transmise à une des instances du Target Group, qui renvoie la page web contenant l’instance-id.




### Partie 2 – Création de l’AMI personnalisée
Une première instance EC2 de base a été lancée à partir de l’AMI standard Amazon Linux 2, en type t2.micro.
 Après connexion en SSH, les paquets ont été mis à jour puis le serveur Apache HTTP a été installé, démarré et activé au démarrage du système (yum update, yum install httpd, systemctl start et enable).
Un script shell /var/www/html/metadata.sh a été créé afin de récupérer un token IMDSv2 et de l’utiliser pour interroger le service de métadonnées EC2.
 Le script récupère l’instance-id de la machine via l’URL des métadonnées et l’écrit dans /var/www/html/index.html, qui est ensuite servi par Apache.



Requête HTTP vers l’endpoint latest/meta-data/instance-id.


Redirection de la sortie vers le fichier index.html du serveur web.


Pour automatiser l’exécution, une tâche cron a été ajoutée avec sudo crontab -e en utilisant la syntaxe @reboot afin de lancer le script à chaque démarrage de l’instance.
 Ainsi, à chaque reboot ou à chaque nouvelle instance dérivée de cette image, la page index.html est régénérée avec l’ID de l’instance en cours.
Une fois tout vérifié (Apache opérationnel, script fonctionnel, page accessible en HTTP), une Amazon Machine Image personnalisée ILF-AMI-WEB a été créée à partir de cette instance.
 Cette AMI sert de base standard pour toutes les instances web du TP, ce qui garantit une configuration homogène.



### Partie 3 – Première instance EC2 et configuration des Security Groups
Un Security Group ILF-SecurityGroup-EC2 a été créé pour les instances web.
 Les règles sont les suivantes :
Entrant SSH (port 22) autorisé uniquement depuis l’adresse IP publique utilisée pour l’administration (IP Ynov).


Entrant HTTP (port 80) autorisé uniquement en provenance du Security Group du Load Balancer ILF-SecurityGroup-LB.


Cette configuration garantit que le serveur web n’est pas directement exposé à Internet, mais uniquement accessible via le Load Balancer, tout en gardant l’accès SSH restreint à l’administrateur.
À partir de l’AMI personnalisée, une instance ILF-Instance1 de type t2.micro a été créée en utilisant ce Security Group.
 Après démarrage, le serveur Apache et le script de métadonnées se sont exécutés automatiquement, et une connexion depuis le Load Balancer a permis d’afficher la page contenant l’instance-id de ILF-Instance1.
Pour les besoins du débogage, l’accès HTTP direct depuis mon IP a été temporairement autorisé afin de vérifier le bon fonctionnement du script et d’Apache.
 Une fois la validation faite, la règle a été resserrée pour ne laisser passer que le trafic provenant du Security Group du Load Balancer, conformément aux consignes de sécurité du TP.



### Partie 4 – Installation de l’AWS CLI et ajout des autres instances
Sur le poste local, l’AWS CLI a été installée puis configurée avec les identifiants fournis pour le TP, la région par défaut étant eu-west-1.
 Cette interface a été utilisée pour créer une seconde instance XXX_Instance2 (et éventuellement une troisième pour des tests supplémentaires) à partir de l’AMI ILF-AMI-WEB, en réutilisant le Security Group ILF-SecurityGroup-EC2.
La commande de création a spécifié l’AMI, le type t2.micro, le Security Group, la paire de clés SSH et le nombre d’instances souhaité.
 Une fois les nouvelles instances disponibles, elles ont été ajoutées au Target Group ILF-TargetGroup grâce à une commande d’enregistrement des cibles qui associe chaque instance-id au port 80 dans le Target Group.


En pratique, seule l’instance créée via l’AWS CLI (ILF-Instance2) était effectivement enregistrée dans le Target Group ILF-TargetGroup lors des derniers tests, tandis que ILF-Instance1 n’y figurait plus.
Lorsque j’appelais le DNS du Load Balancer dans le navigateur, la page affichait donc toujours le même instance-id, ce qui est logique puisque tout le trafic HTTP était dirigé vers cette unique instance présente dans le Target Group, malgré la présence de plusieurs instances EC2 dans la console.

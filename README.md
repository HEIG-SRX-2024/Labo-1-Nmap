# HEIGVD - Sécurité des Réseaux - 2024
# Laboratoire n°1 - Port Scanning et initiation à Nmap

[Introduction](#introduction)\
[Auteurs](#auteurs)\
[Fichiers nécessaires](#fichiers-nécessaires)\
[Rendu](#rendu)\
[Le réseau de test](#le-réseau-de-test)\
[Infrastructure virtuelle](#infrastructure-virtuelle)\
[Connexion à l’infrastructure par OpenVPN](#connexion-à-linfrastructure-par-wireguard)\
[Réseau d’évaluation](#réseau-dévaluation)\
[Scanning avec Nmap](#scanning-avec-nmap)\
[Scanning du réseau (découverte de hôtes)](#scanning-du-réseau-découverte-de-hôtes)\
[Scanning de ports](#scanning-de-ports)\
[Identification de services et ses versions](#identification-de-services-et-ses-versions)\
[Détection du système d’exploitation](#détection-du-système-dexploitation)\
[Vulnérabilités](#vulnérabilités)

# Introduction

Toutes les machines connectées à un LAN (ou WAN, VLAN, VPN, etc…) exécutent des services qui « écoutent » sur certains ports. Ces services sont des logiciels qui tournent dans une boucle infinie en attendant un message particulier d’un client (requête). Le logiciel agit sur la requête ; on dit donc qu’il « sert ».

Le scanning de ports est l’une des techniques les plus utilisées par les attaquants. Ça permet de découvrir les services qui tournent en attendant les clients. L’attaquant peut souvent découvrir aussi la version du logiciel associée à ce service, ce qui permet d’identifier d'éventuelles vulnérabilités.

Dans la pratique, un port scan n’est plus que le fait d’envoyer un message à chaque port et d’en examiner la réponse. Plusieurs types de messages sont possibles et/ou nécessaires. Si le port est ouvert (un service tourne derrière en attendant des messages), il peut être analysé pour essayer de découvrir les vulnérabilités associées au service correspondant.

## Auteurs

Ce texte est basé sur le fichier préparé par Abraham Ruginstein Scharf dans le cadre du
cours Sécurité des Réseaux (SRX) à l'école HEIG/VD, Suisse.
Il a été travaillé et remis en forme pour passer dans un github classroom par
Linus Gasser (@ineiti) du C4DT/EPFL.
L'assistant pour le cours SRX de l'année 2024 est Lucas Gianinetti (@LucasGianinetti).

## Fichiers nécessaires
Vous recevrez par email tous les fichiers nécessaires pour se connecter à l'infrastructure de ce laboratoire.

## Rendu
Ce laboratoire ne sera ni corrigé ni évalué.
Mais je vous conseille quand même de faire une mise à jour de votre répo avec les réponses.
C'est un bon exercice pour le labo-02 qui sera corrigé et noté.

# Le réseau de test

## Infrastructure virtuelle 
Durant ce laboratoire, nous allons utiliser une infrastructure virtualisée. Elle comprend un certain nombre de machines connectées en réseau avec un nombre différent de services.

Puisque le but de ce travail pratique c’est de découvrir dans la mesure du possible ce réseau, nous ne pouvons pas vous en donner plus de détails ! 

Juste un mot de précaution: vous allez aussi voir tous les autres ordinateurs des étudiants qui se connectent au réseau.
C'est voulu, afin que vous puissiez aussi faire des scans sur ceux-ci.
**Par contre, il est formellement interdit de lancer quelconque attaque sur un ordinateur d'un des élèves!**
Si on vous demande de vous attaquer aux machines présentes dans l'infrastructure de test et que vous arrivez à en sortir, veuillez contacter immédiatement le prof ou l'assistant pour récolter des points bonus. Ce n'est pas prévu - mais cela peut arriver :)

## Connexion à l’infrastructure par WireGuard

Notre infrastructure de test se trouve isolée du réseau de l’école. L’accès est fourni à travers une connexion WireGuard.

La configuration de WireGuard varie de système en système. Cependant, dans tous les cas, l’accès peut être géré par un fichier de configuration qui contient votre clé privée ainsi que la clé publique du serveur.

Il est vivement conseillé d’utiliser Kali Linux pour ce laboratoire. WireGuard est déjà préinstallé sur Kali.
Mais ça marche aussi très bien directement depuis un ordinateur hôte - en tout cas j'ai testé Windows et Mac OSX.
Vous trouvez les clients WireGuard ici: https://www.wireguard.com/install/

Vous trouverez dans l’email reçu un fichier de configuration WireGuard personnalisé pour vous (chaque fichier est unique) ainsi que quelques informations relatives à son utilisation. Le fichier contient un certificat et les réglages corrects pour vous donner accès à l’infra.

Une fois connecté à l’infrastructure, vous recevrez une adresse IP correspondante au réseau de test.

Pour vous assurer que vous êtes connecté correctement au VPN, vous devriez pouvoir pinger l’adresse 10.1.2.1 ou 10.1.1.2.

### Configuration Kali Linux

Pour l'installation dans Kali-Linux, il faut faire la chose suivante:

```bash
sudo -i
apt update
apt install -y wireguard resolvconf
vi /etc/wireguard/wg0.conf # copier le contenu de peerxx.conf
#Connexion au VPN
wg-quick up wg0
#Déconnexion du VPN
wg-quick down wg0
```

### Réseau d’évaluation

Le réseau que vous allez scanner est le 10.1.1.0/24 - le réseau 10.1.2.0/24 est le réseau WireGuard avec tous les
ordinateurs des élèves. On va essayer de le scanner vite fait, mais **INTERDICTION DE FAIRE DU PENTEST SUR CES MACHINES**!

### Distribution des fichiers de configuration

Pour simplifier ce labo, je vous ai directement envoyé les fichiers de configuration.
Mais dans un environnement où on ne fait pas forcément confiance au serveur, ni à la personne qui distribue les
fichiers, ceci n'est pas une bonne pratique.

Quels sont les vecteurs d'attaque pour cette distribution?
Qui est une menace?
**LIVRABLE: texte**

Comment est-ce qu'il faudrait procéder pour palier à ces attaques?
Qui devrait envoyer quelle information à qui? Et dans quel ordre?
**LIVRABLE: texte**

# Scanning avec Nmap

Nmap est considéré l’un des outils de scanning de ports les plus sophistiqués et évolués. Il est développé et maintenu activement et sa documentation est riche et claire. Des centaines de sites web contiennent des explications, vidéos, exercices et tutoriels utilisant Nmap.

## Scanning du réseau (découverte de hôtes)

Le nom « Nmap » implique que le logiciel fut développé comme un outil pour cartographier des réseaux (Network map). Comme vous pouvez l’imaginer, cette fonctionnalité est aussi attirante pour les professionnels qui sécurisent les réseaux que pour ceux qui les attaquent.

Avant de pouvoir se concentrer sur les services disponibles sur un serveur en particulier et ses vulnérabilités, il est utile/nécessaire de dresser une liste d’adresses IP des machines présentes dans le réseau. Ceci est particulièrement important, si le réseau risque d’avoir des centaines (voir des milliers) de machines connectées. En effet, le scan de ports peut prendre longtemps tandis que la découverte de machines « vivantes », est un processus plus rapide et simple. Il faut quand-même prendre en considération le fait que la recherche simple d'hôtes ne retourne pas toujours la liste complète de machines connectées.

Nmap propose une quantité impressionnante de méthodes de découverte de hôtes. L’utilisation d’une ou autre méthode dépendra de qui fait le scanning (admin réseau, auditeur de sécurité, pirate informatique, amateur, etc.), pour quelle raison le scanning est fait et quelle infrastructure est présente entre le scanner et les cibles.

**Questions**

> a.	Quelles options sont proposées par Nmap pour la découverte des hôtes ? Servez-vous du menu « help » de Nmap (nmap -h), du manuel complet (man nmap) et/ou de la documentation en ligne.   

**LIVRABLE: texte** :

> b.	Essayer de dresser une liste des hôtes disponibles dans le réseau en utilisant d’abord un « ping scan » (No port scan) et ensuite quelques autres méthodes de scanning (dans certains cas, un seul type de scan pourrait rater des hôtes).

Adresses IP trouvées :

**LIVRABLE: texte** :

> c. Avez-vous constaté des résultats différents en utilisant les différentes méthodes ? Pourquoi pensez-vous que ça pourrait être le cas ?

**LIVRABLE: texte** :

> d. Quelles options de scanning sont disponibles si vous voulez être le plus discret possible ?

**LIVRABLE: texte** :

## Scanning de ports

Il y a un total de 65'535 ports TCP et le même nombre de ports UDP, ce qui rend peu pratique une analyse de tous les ports, surtout sur un nombre important de machines. 

N’oublions pas que le but du scanning de ports est la découverte de services qui tournent sur le système scanné. Les numéros de port étant typiquement associés à certains services connus, une analyse peut se porter sur les ports les plus « populaires ».

Les numéros des ports sont divisés en trois types :

-	Les ports connus : du 0 au 1023
-	Les ports enregistrés : du 1024 au 49151
-	Les ports dynamiques ou privés : du 49152 au 65535

**Questions**
> e.	Complétez le tableau suivant :

**LIVRABLE: tableau** :

| Port	| Service	| Protocole (TCP/UDP)   |
| :---: | :---:     | :---:                 |
| 20/21	|           |                       |
| 22	|           |                       |
| 23	|           |                       |
| 25	|           |                       |
| 53	|           |                       |
| 67/68	|           |                       |
| 69	|           |                       |
| 80	|           |                       |
| 110	|           |                       |
| 443	|           |                       |
| 3306	|           |                       |

> f.	Par défaut, si vous ne donnez pas d’option à Nmap concernant les port, quelle est la politique appliquée par Nmap pour le scan ? Quels sont les ports qui seront donc examinés par défaut ? Servez-vous de la documentation en ligne pour trouver votre réponse.

**LIVRABLE: texte** :


>g.	Selon la documentation en ligne de Nmap, quels sont les ports TCP et UDP le plus souvent ouverts ? Quels sont les services associés à ces ports ?   

**LIVRABLE: texte** :


>h.	Dans les commandes Nmap, de quelle manière peut-on cibler un numéro de port spécifique ou un intervalle de ports ? Servez-vous du menu « help » de Nmap (nmap -h), du manuel complet (man nmap) et/ou de la documentation en ligne.   

**LIVRABLE: texte** :


>i.	Quelle est la méthode de scanning de ports par défaut utilisée par Nmap si aucune option n’est donnée par l’utilisateur ?

**LIVRABLE: texte** :


>j.	Compléter le tableau suivant avec les options de Nmap qui correspondent à chaque méthode de scanning de port :

**LIVRABLE: tableau** :

| Type de scan	| Option nmap   |
| :---:         | :---:         |
| TCP (connect) |               |
| TCP SYN       |               |
| TCP NULL      |               |
| TCP FIN       |               |
| TCP XMAS      |               |
| TCP idle (zombie) |               |	
| UDP           |               |

>k.	Lancer un scan du réseau entier utilisant les méthodes de scanning de port TCP, SYN, NULL et UDP. Y a-t-il des différences au niveau des résultats pour les scans TCP ? Si oui, lesquelles ? Avez-vous un commentaire concernant le scan UDP ?

**LIVRABLE: texte** :

> l.	Ouvrir Wireshark, capturer sur votre interface réseau et relancer un scan TCP (connect) sur une seule cible spécifique. Observer les échanges entre le scanner et la cible. Lancer maintenant un scan SYN en ciblant spécifiquement la même machine précédente. Identifier les différences entre les deux méthodes et les contraster avec les explications théoriques données en cours. Montrer avec des captures d’écran les caractéristiques qui définissent chacune des méthodes.

Capture pour TCP (connect)

**LIVRABLE: capture d'écran** :

Capture pour SYN :

**LIVRABLE: cpature d'écran** :

>m.	Quelle est l’adresse IP de la machine avec le plus grand nombre de services actifs ? 

**LIVRABLE: texte** :

## Identification de services et ses versions

Le fait de découvrir qu’un certain port est ouvert, fermé ou filtré n’est pas tellement utile ou intéressant sans connaître son service et son numéro de version associé. Cette information est cruciale pour identifier d'éventuelles vulnérabilités et pour pouvoir tester si un exploit est réalisable ou pas.

**Questions**

>n.	Trouver l’option de Nmap qui permet d’identifier les services (servez-vous du menu « help » de Nmap (nmap -h), du manuel complet (man nmap) et/ou de la documentation en ligne). Utiliser la commande correcte sur l’un des hôtes que vous avez identifiés avec des ports ouverts (10.1.1.10 vivement recommandé…). Montrer les résultats.   

Résultat du scan d’identification de services :

**LIVRABLE: texte** :

## Détection du système d’exploitation
Nmap possède une base de données contenant plus de 2600 systèmes d’exploitation différents. La détection n’aboutit pas toujours mais quand elle fonctionne, Nmap est capable d’identifier le nom du fournisseur, l’OS, la version, le type de dispositif sur lequel l’OS tourne (console de jeux, routeur, switch, dispositif générique, etc.) et même une estimation du temps depuis le dernier redémarrage de la cible.

**Questions** 

>o.	Chercher l’option de Nmap qui permet d’identifier le système d’exploitation (servez-vous du menu « help » de Nmap (nmap -h), du manuel complet (man nmap) et/ou de la documentation en ligne). Utiliser la commande correcte sur la totalité du réseau. Montrer les résultats.   

Résultat du scan d’identification du système d’exploitation :

**LIVRABLE: texte** :

>p. Avez-vous trouvé l’OS de toutes les machines ? Sinon, en utilisant l’identification de services, pourrait-on se faire une idée du système de la machine ?

**LIVRABLE: texte** :

>q. Vous voyez une différence entre les machines misent à disposition pour le cours et les machines connectées au réseau.
Expliquez pourquoi cette différence est là.

**LIVRABLE: texte** :

## Vulnérabilités 
Servez-vous des résultats des scans d’identification de services et de l’OS pour essayer de trouver des vulnérabilités. Vous pouvez employer pour cela l’une des nombreuses bases de données de vulnérabilités disponibles sur Internet. Vous remarquerez également que Google est un outil assez puissant pour vous diriger vers les bonnes informations quand vous connaissez déjà les versions des services et des OS.

**IL EST INTERDIT DE S'ATTAQUER AUX ORDINATEURS DES AUTRES éTUDIANTS!**

**Questions**

>r.	Essayez de trouver des services vulnérables sur la machine que vous avez scanné avant (vous pouvez aussi le faire sur d’autres machines. Elles ont toutes des vulnérabilités !). 

Résultat des recherches :

**LIVRABLE: texte** :

> Challenge: L’une des vulnérabilités sur la machine 10.1.1.2 est directement exploitable avec rien d’autre que Netcat. Est-ce que vous arrivez à le faire ?

**LIVRABLE: texte** :

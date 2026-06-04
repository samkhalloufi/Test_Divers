# Test_Divers

## Flow Node-RED BACnet/IP pour Milesight UG65

Le fichier `node-red-bacnet-discovery-ug65.json` contient un flow d'import Node-RED 3.0.2 conçu pour une gateway Milesight UG65. Il permet de découvrir un équipement BACnet/IP sur le même réseau local, sans installer de module Node-RED additionnel.

### Ce que fait le flow

- Envoie un `Who-Is` BACnet/IP en broadcast UDP vers `255.255.255.255:47808`.
- Écoute les réponses `I-Am` sur UDP `47808`.
- Enregistre les devices trouvés dans le contexte de flow `bacnetDevices`.
- Lit automatiquement la propriété `objectList` du device détecté.
- Lit ensuite `objectName` pour chaque objet afin de rendre la liste de points exploitable.
- Affiche les résultats dans deux nodes debug : `Devices BACnet trouves` et `Points BACnet trouves`.

### Import et utilisation dans Node-RED 3.0.2 embarqué UG65

1. Ouvrir l'éditeur Node-RED de l'UG65.
2. Aller dans **Menu > Import**.
3. Coller le contenu de `node-red-bacnet-discovery-ug65.json`.
4. Cliquer sur **Import**, puis **Deploy**.
5. Cliquer sur l'injecteur **Who-Is BACnet/IP broadcast**.
6. Ouvrir le panneau debug Node-RED pour lire les devices et les points découverts.


### Correction renforcée pour la découverte BACnet/IP

Après vérification de la logique BACnet/IP, le flow de découverte utilise maintenant une approche plus proche des outils BACnet classiques :

- le node `BACnet UDP in 47808` est placé avant le node d'envoi dans le JSON pour créer d'abord le socket d'écoute ;
- le node `UDP out Who-Is source 47808` envoie depuis le port source local `47808`, afin que les réponses `I-Am` unicast reviennent sur le port écouté ;
- le flow n'utilise plus uniquement le global broadcast `255.255.255.255`, souvent filtré ou problématique ;
- il envoie un `Who-Is` en broadcast dirigé vers `192.168.0.255:47808` ;
- il envoie aussi un `Who-Is` unicast vers le device connu `192.168.0.210:47808`, ce qui permet de tester le device fourni même si le broadcast est filtré.

Si le LAN de l'UG65 n'est pas `192.168.0.0/24`, adapter dans la function `Construire Who-Is` la constante `DIRECTED_BROADCAST` avec le broadcast réel du réseau, par exemple `192.168.1.255` pour un réseau `192.168.1.0/24`.

### Historique des essais de broadcast

La première version arrivait à déclencher la découverte device, mais elle dépendait du comportement de réponse du device après un broadcast global `255.255.255.255`. La version actuelle est plus robuste pour le device fourni : elle combine broadcast dirigé local et Who-Is unicast, tout en forçant le port source local `47808` pour recevoir les `I-Am` unicast.


### Correction pour la découverte des points après `I-Am`

Le device répondait bien au `Who-Is`, donc la partie découverte device était correcte. Le blocage venait de l'étape suivante : les requêtes `ReadProperty` utilisées pour lire `objectList` et `objectName` sortaient par le node `UDP out vers msg.ip/msg.port` sans port source local fixé.

Dans ce cas, Node-RED peut envoyer la requête depuis un port UDP éphémère, alors que le flow écoute uniquement le port BACnet `47808`. Le device répond alors au port source éphémère et la réponse `ReadProperty-ACK` n'arrive jamais dans le node `BACnet UDP in 47808`. Résultat : le device apparaît, mais aucun point n'est découvert.

Le node de lecture a donc été corrigé en `UDP out ReadProperty source 47808` avec `outport: "47808"`. Les `ReadProperty objectList/objectName` partent maintenant du même port source que le socket écouté, et les réponses peuvent être traitées par le parseur.

### Correction conservée pour la découverte des points

Si le device était détecté mais qu'aucun point ne remontait, la cause venait du décodage de la réponse BACnet `ReadProperty-ACK`.

Dans BACnet, la valeur retournée par un `ReadProperty-ACK` est encapsulée dans un tag ouvrant contextuel `[3]` (`0x3e`) et un tag fermant `[3]` (`0x3f`). La première version du flow essayait de décoder directement le tag `0x3e` comme si c'était la valeur applicative de `objectList[0]`. Résultat : le device était bien découvert via `I-Am`, mais le compteur `objectList[0]` n'était jamais lu, donc la boucle de lecture des points ne démarrait pas.

Le flow ignore maintenant correctement le tag ouvrant `[3]`, lit la valeur réelle, puis peut enchaîner la lecture de chaque entrée `objectList[index]` et de son `objectName`.

### Points d'attention sur UG65

- L'UG65 et l'équipement BACnet/IP doivent être dans le même LAN/VLAN, car la découverte utilise un broadcast local.
- Le port UDP `47808` ne doit pas être déjà monopolisé par un autre service BACnet local sur l'UG65.
- Le flow n'utilise que des nodes core Node-RED (`inject`, `function`, `udp in`, `udp out`, `debug`, `comment`) pour rester compatible avec les environnements embarqués ou l'installation de paquets npm est limitée.
- Certains équipements BACnet peuvent refuser une lecture complète si `objectList` est très volumineux. Le flow évite ce problème en lisant d'abord `objectList[0]`, puis chaque entrée une par une.
- Si aucune réponse n'apparaît, vérifier le firewall, le VLAN, le masque réseau et l'activation BACnet/IP côté device.

### Fichier fourni

- `node-red-bacnet-discovery-ug65.json` : flow importable dans Node-RED.

## Flow de lecture des points BACnet connus

Le fichier `node-red-bacnet-read-fixed-points-ug65.json` contient un flow séparé pour lire directement les valeurs `present-value` des trois points fournis sur le device BACnet/IP suivant :

- `deviceId`: `10000` ;
- `IP`: `192.168.0.210` ;
- port BACnet/IP distant : UDP `47808` ;
- port local client utilisé par l'UG65 pour ce flow de lecture : UDP `47809`.

### Points lus

| object-name | object-type ID | object-type | object-instance | propriété lue |
| --- | ---: | --- | ---: | --- |
| `Space Temperature Setpoint BAS|rt-1` | `1` | `ANALOG_OUTPUT` | `6` | `present-value` (`85`) |
| `Space Temperature Setpoint BAS|rt-2` | `1` | `ANALOG_OUTPUT` | `14` | `present-value` (`85`) |
| `Space Temperature Setpoint BAS|rt-3` | `1` | `ANALOG_OUTPUT` | `22` | `present-value` (`85`) |

### Utilisation

1. Importer `node-red-bacnet-read-fixed-points-ug65.json` dans Node-RED.
2. Déployer le flow.
3. Cliquer sur l'injecteur **Lire les 3 consignes maintenant**.
4. Lire les résultats dans le debug **Valeurs BACnet lues**.

### Important sur les ports UDP

Le flow envoie toujours les requêtes vers le device sur le port BACnet/IP standard `192.168.0.210:47808`, mais il utilise maintenant un port local client séparé sur l'UG65 : UDP `47809`.

Ce changement évite les conflits avec le flow de découverte ou avec un service BACnet local déjà attaché à UDP `47808`. La version précédente utilisait aussi `47808` comme port source local ; cela pouvait fonctionner, mais Node-RED 3.0.2 partage les sockets UDP par port et un autre tab pouvait fermer ou capter le socket, ce qui expliquait l'absence de réponse visible malgré un device en ligne.

Si ton équipement BACnet refuse les requêtes venant d'un port source différent de `47808`, modifie dans le flow les deux champs suivants de `47809` vers `47808` : le node `BACnet UDP in client 47809` et le champ `outport` du node `UDP out vers 192.168.0.210:47808 source 47809`. Dans ce cas, désactive tous les autres tabs/services BACnet utilisant UDP/47808, puis fais un **Deploy complet**.

## Flow avec `@halsystems/red-bacnet`

Le fichier `node-red-red-bacnet-ug65.json` contient un flow prêt à importer après installation du module `@halsystems/red-bacnet`.

### Configuration intégrée

- Gateway UG65 / interface BACnet locale : `192.168.0.150`.
- Port BACnet local : UDP `47808`.
- Broadcast BACnet : `192.168.0.255`.
- Device cible : `deviceId 10000`, IP `192.168.0.210`.

### Utilisation

1. Installer `@halsystems/red-bacnet` dans **Manage palette** ou en SSH avec `npm install --production @halsystems/red-bacnet` dans le dossier utilisateur Node-RED.
2. Redémarrer Node-RED si les nodes `bacnet client`, `discover device`, `discover point` et `read point` n'apparaissent pas.
3. Importer `node-red-red-bacnet-ug65.json`.
4. Déployer le flow.
5. Lancer les injecteurs dans l'ordre :
   - **1 - Discover device 10000** ;
   - **2 - Discover points device connu** ;
   - **3 - Read 3 points connus**.

Le flow reprend les noms et structures de messages de l'exemple officiel `@halsystems/red-bacnet`, en les adaptant à ton UG65 et à ton device `10000@192.168.0.210`.

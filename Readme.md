<h1 align="center"> Programmateur horaire ESP32  Version 3.3  </h1> 

<h2 align="center">N relais, interface web, écran OLED, OTA, programmation à la minute près, réseau Wi-Fi de secours et réglage manuel de l'heure</h2>

![Platform](https://img.shields.io/badge/Platform-ESP32-green)
![Framework](https://img.shields.io/badge/Framework-Arduino-blue)
![Status](https://img.shields.io/badge/Status-Active-green)
![Release](https://img.shields.io/badge/Release-v3.3.Secours-orange)


Ce projet transforme un **ESP32** en programmateur horaire connecté capable de piloter plusieurs relais indépendants.

Le programme utilisé est :

`Programmateur_horaire_ESP32_V3_Secours.ino`

Fonctions principales :

- pilotage d'un nombre configurable de relais ;
- mode **AUTO** avec programmation horaire ;
- mode **MANUEL** avec forçage ON/OFF ;
- jusqu'à `MAX_PLAGES` plages horaires par relais ;
- horaires réglables **à la minute près** ;
- plages pouvant traverser minuit ;
- effacement complet des plages possible ;
- sauvegarde des réglages dans la mémoire flash **NVS** ;
- interface Web embarquée directement dans l'ESP32 ;
- interface adaptée aux smartphones ;
- accès par adresse IP et par **mDNS** (`richardv.local`) ;
- affichage sur OLED SSD1306 128×64 ;
- boutons physiques optionnels avec anti-rebond ;
- connexion automatique au meilleur réseau Wi-Fi connu ;
- reconnexion Wi-Fi non bloquante ;
- recherche périodique d'un réseau connu offrant un meilleur signal ;
- mise à jour du firmware par **OTA** ;
- affichage de la puissance Wi-Fi en dBm et en pourcentage ;
- validation des horaires avant sauvegarde ;
- conservation des réglages lors d'une simple recompilation ou mise à jour OTA ;
- réseau Wi-Fi de secours **`ESP32_Secours`** ouvert automatiquement quand la box est injoignable (accès smartphone sur `http://192.168.5.1`) ;
- **réglage manuel de l'heure** depuis la page Web (heure du smartphone en un clic, ou saisie manuelle) ;
- fonctionnement sans box au démarrage (plus de redémarrage en boucle) et lecture de l'heure non bloquante ;
- **portail captif** et serveur DNS sur le réseau de secours (`richardv.local` y fonctionne aussi) ;
- **Internet simulé** : le smartphone reste sur le réseau de secours **sans avoir à couper les données mobiles**.

### Nouveautés des versions 3.x

| Nouveauté | Intérêt |
|---|---|
| Réseau de secours `ESP32_Secours` | Piloter l'ESP32 avec un smartphone pendant une coupure de la box |
| Réglage manuel de l'heure | Garder une programmation horaire juste sans Internet ni NTP |
| Plus de `ESP.restart()` sans box | L'ESP32 reste utilisable et accessible pendant la coupure |
| `getLocalTime()` non bloquant | Sans NTP, l'ESP32 ne se fige plus 5 s à chaque seconde |
| Infos système enrichies | État du réseau de secours et origine de l'heure |
| **V3.1** – Protection `ESP_xxxxxx` | Plus jamais de réseau par défaut ouvert sans mot de passe |
| **V3.2** – Serveur DNS + portail captif | `richardv.local` et 192.168.5.1 fiables sur le réseau de secours |
| **V3.3** – Internet simulé | Plus besoin de couper les données mobiles du smartphone |

---

# 2. Architecture générale

Le fonctionnement peut être résumé ainsi :

```text
                         ┌─────────────────────┐
                         │       ESP32         │
                         │                     │
                         │  Programme horaire  │
                         │        + NVS        │
                         └──────────┬──────────┘
                                    │
             ┌──────────────────────┼──────────────────────┐
             │                      │                      │
             ▼                      ▼                      ▼
       Interface Web             OLED SSD1306          Boutons physiques
       Smartphone/PC              128 x 64             optionnels
   (via la box, ou via le
    réseau ESP32_Secours)
             │                      │                      │
             └──────────────────────┼──────────────────────┘
                                    │
                                    ▼
                              GPIO des relais
                                    │
                    ┌───────────────┼───────────────┐
                    ▼               ▼               ▼
                  Relais 1        Relais 2       Relais N
```

Le navigateur ne contient pas la programmation en dur : il demande la configuration à l'ESP32 avec `/get-config`, puis construit dynamiquement l'interface.

---

# 3. Configuration actuelle du programme

La configuration livrée avec le fichier est :

| Programmateur | Fonction | GPIO relais | GPIO bouton | Mode par défaut | Plages par défaut |
|---|---|---:|---:|---|---|
| 1 | Cuisine | GPIO32 | GPIO14 | AUTO | 06:32-08:10, 09:00-13:15 |
| 2 | Portail | GPIO33 | GPIO16 | AUTO | 07:05-09:10, 17:08-19:33 |
| 3 | Relais 3 | GPIO25 | GPIO17 | AUTO | aucune |
| 4 | Relais 4 | GPIO26 | GPIO18 | AUTO | 23:17-01:52 |

Les quatre relais sont actuellement configurés :

```cpp
relayActiveHigh = true
```

Cela signifie :

```text
GPIO HIGH → relais activé
GPIO LOW  → relais désactivé
```

Si votre carte relais fonctionne en logique inverse, mettre :

```cpp
false
```

pour le relais concerné.

---

# 4. Matériel nécessaire

## 4.1 Éléments principaux

- ESP32 compatible Arduino ;
- module(s) relais compatibles avec les niveaux logiques de l'ESP32 ;
- alimentation adaptée à l'ESP32 et aux relais ;
- écran OLED SSD1306 I2C 128×64, adresse `0x3C` ;
- boutons poussoirs optionnels ;
- câblage adapté.

## 4.2 Attention au niveau logique

Les GPIO de l'ESP32 fonctionnent en logique **3,3 V**.

Ne pas appliquer directement du 5 V sur une GPIO.

Selon le module relais utilisé, vérifier :

- tension de commande ;
- courant demandé par l'entrée ;
- compatibilité avec un signal 3,3 V ;
- logique active HIGH ou active LOW.

---

# 5. Câblage OLED

L'écran OLED SSD1306 utilisé par le programme est :

```text
Résolution : 128 × 64
Interface  : I2C
Adresse    : 0x3C
```

Câblage prévu :

| OLED | ESP32 |
|---|---|
| VCC | alimentation adaptée au module |
| GND | GND |
| SDA | GPIO21 |
| SCL | GPIO22 |

Dans le programme :

```cpp
#define SCREEN_WIDTH   128
#define SCREEN_HEIGHT  64
#define OLED_RESET     -1
#define SCREEN_ADDRESS 0x3C
```

Le bus I2C est initialisé par :

```cpp
Wire.begin();
```

---

# 6. Câblage des relais

Configuration actuelle :

| Relais | GPIO ESP32 | Logique |
|---|---:|---|
| Relais 1 | GPIO32 | actif HIGH |
| Relais 2 | GPIO33 | actif HIGH |
| Relais 3 | GPIO25 | actif HIGH |
| Relais 4 | GPIO26 | actif HIGH |

La fonction centrale utilisée pour commander un relais est :

```cpp
ecrireRelais(const Programmateur &p, bool etat)
```

Elle tient compte individuellement de `relayActiveHigh`.

Cette centralisation évite d'avoir des `digitalWrite()` incompatibles dispersés dans le programme.

---

# 7. Boutons physiques

Les boutons sont optionnels.

Configuration actuelle :

| Bouton | Programmateur | GPIO |
|---|---|---:|
| BP1 | Relais 1 | GPIO14 |
| BP2 | Relais 2 | GPIO16 |
| BP3 | Relais 3 | GPIO17 |
| BP4 | Relais 4 | GPIO18 |

Câblage :

```text
GPIO bouton ───── bouton poussoir ───── GND
```

Le programme utilise :

```cpp
pinMode(pinBP, INPUT_PULLUP);
```

Il n'est donc normalement pas nécessaire d'ajouter une résistance externe de pull-up.

### Fonctionnement

Un appui bref :

1. inverse l'état du relais ;
2. passe automatiquement le relais en mode MANUEL ;
3. sauvegarde le nouvel état dans la NVS.

Un relais forcé en manuel n'est plus piloté par ses horaires tant qu'il reste en mode MANUEL.

---

# 8. Ajouter ou supprimer un relais

La configuration principale se trouve dans :

```cpp
Programmateur programmateurs[] = {
```

Exemple :

```cpp
{ "5", "Programmation 5", "Relais 5", "#a78bfa",
  27, true, "12:00-22:00", true, false, -1 },
```

Les champs sont :

```text
id
nom
sousNom
couleur
GPIO relais
relayActiveHigh
plagesDefaut
modeAuto
relayState
GPIO bouton
```

### Exemple sans bouton physique

```cpp
{ "5", "Programmation 5", "Relais 5", "#a78bfa",
  27, true, "12:00-22:00", true, false, -1 },
```

`-1` signifie :

```text
aucun bouton physique
```

La page Web, les routes HTTP, la NVS et l'affichage OLED utilisent automatiquement le nombre de lignes présentes dans `programmateurs[]`.

---

# 9. GPIO à utiliser avec prudence

Le programme fournit notamment les GPIO suivants comme sorties possibles sur un ESP32 DevKit classique :

```text
4, 5, 13, 14, 16, 17, 18, 19,
21, 22, 23, 25, 26, 27, 32, 33
```

Mais :

- GPIO21 = SDA I2C ;
- GPIO22 = SCL I2C ;
- GPIO34 à GPIO39 = entrées uniquement ;
- les GPIO de boot doivent être utilisés avec prudence ;
- les GPIO déjà utilisés par d'autres périphériques ne doivent pas être réutilisés.

Toujours vérifier le brochage exact de votre modèle d'ESP32.

---

# 10. Nombre maximal de plages horaires

Le nombre maximal est défini ici :

```cpp
const int MAX_PLAGES = 6;
```

Actuellement :

```text
6 plages maximum par relais
```

Pour passer à 8 :

```cpp
const int MAX_PLAGES = 8;
```

La structure, la NVS et l'interface Web sont conçues pour suivre automatiquement cette valeur.

---

# 11. Format des plages horaires

Les horaires sont exprimés à la minute près.

Exemples valides :

```text
06:32-08:10
09:00-13:15
18:45-22:30
```

Plusieurs plages sont séparées par une virgule :

```text
06:32-08:10,09:00-13:15,18:45-22:30
```

---

# 12. Plage traversant minuit

Le programme accepte directement :

```text
23:17-01:52
```

Cela signifie :

```text
23:17 → 24:00
+
00:00 → 01:52
```

Il n'est pas nécessaire de créer deux plages.

La fonction `plageEstActive()` traite automatiquement ce cas.

---

# 13. Utilisation de 24:00

Le moteur interne autorise :

```text
24:00
```

comme heure de fin.

Exemple :

```text
06:00-24:00
```

signifie :

```text
06:00 → minuit
```

`24:00` est stocké comme :

```text
1440 minutes
```

Le champ HTML `input type="time"` ne gérant pas correctement `24:00` dans tous les navigateurs, l'interface Web utilise une gestion spécifique pour cette valeur.

---

# 14. Effacer toutes les plages

Une programmation vide est autorisée :

```text
plages=""
```

Cela signifie :

```text
aucune plage horaire
```

En mode AUTO, le relais reste donc éteint, sauf autre logique spécifique du programme.

Important : une liste vide sauvegardée dans la NVS est distinguée d'une absence totale de configuration.

---

# 15. Validation des horaires

Avant d'être enregistrée, une nouvelle programmation est vérifiée par :

```cpp
textePlagesValide()
```

Le programme vérifie notamment :

- présence du séparateur `-` ;
- validité des heures ;
- validité des minutes ;
- respect du nombre maximum de plages ;
- impossibilité d'avoir une plage de durée nulle.

Exemple refusé :

```text
08:00-08:00
```

Exemple accepté :

```text
08:00-10:00
```

Si une programmation est invalide, l'ancienne programmation reste conservée.

---

# 16. Interface Web

L'interface HTML/CSS/JavaScript est intégrée directement dans le fichier `.ino`.

Il n'est pas nécessaire d'utiliser :

- carte SD ;
- LittleFS ;
- SPIFFS ;
- fichier HTML externe.

La page est placée dans :

```cpp
const char index_html[] PROGMEM = R"rawliteral(
...
)rawliteral";
```

L'interface s'adapte automatiquement au nombre de relais.

---

# 17. Accès à l'interface

Après connexion Wi-Fi, l'ESP32 est normalement accessible par :

```text
http://richardv.local
```

et également par son adresse IP.

Exemple :

```text
http://192.168.1.50
```

Le nom mDNS est défini par :

```cpp
const char* hostname = "richardv";
```

Si vous changez cette ligne :

```cpp
const char* hostname = "monesp32";
```

l'adresse devient :

```text
http://monesp32.local
```

En cas de coupure de la box, l'interface reste accessible par le réseau de secours :

```text
Réseau Wi-Fi : ESP32_Secours
Adresse      : http://192.168.4.1   (ou http://richardv.local depuis la V3.2)
```

---

# 18. Compatibilité smartphone

La page Web utilise notamment :

```html
<meta name="viewport"
      content="width=device-width, initial-scale=1, viewport-fit=cover">
```

L'interface est conçue pour une largeur d'environ 430 px et s'adapte aux écrans de smartphones.

Elle peut être utilisée depuis :

- Android ;
- iPhone/iPad ;
- Windows ;
- Linux ;
- macOS ;

à condition que l'appareil soit sur le réseau local approprié.

---

# 19. Authentification Web

### Aucune authentification sur l'interface Web

Dans cette version, **aucune route HTTP ne demande d'identifiant ni de mot de passe** :

```text
/save          modification des plages horaires
/toggle-mode   bascule AUTO / MANUEL
/force-state   forçage ON / OFF
/reset-auto    retour de tous les relais en AUTO
/set-time      réglage manuel de l'heure
```

Toutes les commandes sont donc utilisables directement depuis le smartphone, sans fenêtre de connexion.

### Ce qui protège l'accès

L'accès repose uniquement sur l'accès au réseau Wi-Fi :

- **réseau de la box** : seuls les appareils connectés à votre Wi-Fi local peuvent ouvrir la page ;
- **réseau de secours `ESP32_Secours`** : protégé par son mot de passe WPA2 `SECRET_AP_PASS`.

> ⚠️ Toute personne connectée à l'un de ces réseaux peut commander les relais (portail, garage…). Ne pas donner le mot de passe Wi-Fi à des invités non souhaités et personnaliser `SECRET_AP_PASS`.

### Rôle de `SECRET_OTA_PASSWORD`

`SECRET_OTA_PASSWORD` sert **uniquement** à protéger la mise à jour du firmware par OTA. Il n'est pas utilisé par l'interface Web.

---

# 20. Attention à la sécurité

Le serveur Web fonctionne en :

```text
HTTP
```

et non en HTTPS.

Les échanges ne sont pas chiffrés et l'interface Web n'a pas de mot de passe : la sécurité repose sur le réseau Wi-Fi lui-même.

Le projet est destiné en priorité à un **réseau local de confiance**.

Éviter d'exposer directement le port 80 de l'ESP32 sur Internet.

---

# 21. Fichier arduino_secrets.h

Le fichier :

```text
arduino_secrets.h
```

n'est normalement pas inclus dans le dépôt ou dans le partage public du programme.

Il doit contenir les informations Wi-Fi, le mot de passe OTA et le mot de passe du réseau de secours.

Exemple de structure :

```cpp
#define SECRET_SSID  "Nom_du_WiFi_1"
#define SECRET_PASS  "Mot_de_passe_WiFi_1"

#define SECRET_SSID2 "Nom_du_WiFi_2"
#define SECRET_PASS2 "Mot_de_passe_WiFi_2"

#define SECRET_OTA_PASSWORD "Mot_de_passe_OTA"

//  mot de passe du réseau de secours ESP32_Secours (8 caractères minimum)
#define SECRET_AP_PASS "Mot_de_passe_secours"
```

Si `SECRET_AP_PASS` n'est pas défini, c'est la valeur écrite dans `AP_PASS` du programme qui est utilisée. **Il est vivement conseillé de définir `SECRET_AP_PASS`.**

Si un troisième réseau est utilisé :

```cpp
#define SECRET_SSID3 "Nom_du_WiFi_3"
#define SECRET_PASS3 "Mot_de_passe_WiFi_3"
```

Le troisième réseau est pris en compte par :

```cpp
#ifdef SECRET_SSID3
```

---

# 22. Connexion Wi-Fi

Le programme peut connaître plusieurs réseaux Wi-Fi.

Il commence par scanner les réseaux visibles.

Il ne conserve que ceux qui sont présents dans :

```cpp
knownNetworks[]
```

Il sélectionne ensuite le réseau connu offrant le meilleur RSSI.

Le programme utilise une gestion non bloquante pour les recherches ultérieures.

###  Aucun réseau connu au démarrage
1. affiche `Box injoignable` sur l'OLED avec le nom du réseau de secours et l'adresse `http://192.168.4.1` ;
2. ouvre immédiatement le réseau `ESP32_Secours` ;
3. continue à piloter les relais, les boutons et la page Web ;
4. recherche la box en tâche de fond.

---

# 23. Reconnexion Wi-Fi

Si le Wi-Fi est perdu :

1. la boucle principale continue ;
2. l'ESP32 ne reste pas bloqué dans une longue attente ;
3. une nouvelle recherche est lancée ;
4. les réseaux connus sont évalués ;
5. une reconnexion est tentée.

La tentative est espacée dans le temps afin d'éviter une boucle de reconnexion permanente.

| Situation | Intervalle entre deux recherches de la box |
|---|---:|
| Pas de réseau de secours ouvert | 10 s |
| Réseau de secours ouvert, aucun appareil connecté | 30 s |
| Smartphone connecté au réseau de secours | 3 min |

Chaque recherche (scan Wi-Fi) perturbe brièvement le réseau de secours : l'intervalle est donc allongé pour ne pas gêner le smartphone.

---

# 24. Recherche d'un meilleur réseau

Même lorsque l'ESP32 est déjà connecté, le programme vérifie périodiquement si un autre réseau connu offre un signal sensiblement meilleur.

Une marge RSSI est utilisée pour éviter des changements incessants entre deux réseaux proches.

La constante utilisée est :

```cpp
const int RSSI_SWITCH_MARGIN = 8;
```

Cela correspond à une marge d'environ 8 dBm.

---

# 25. Affichage du signal Wi-Fi

La page Web affiche :

- RSSI en dBm ;
- estimation en pourcentage.

Exemple :

```text
-52 dBm
```

Le pourcentage est une estimation destinée à rendre la lecture plus intuitive.

Le dBm reste la valeur technique la plus utile pour comparer deux signaux.

---

# 26. NTP et heure française

Le programme utilise NTP pour synchroniser l'horloge interne :

```cpp
configTzTime(
  TZ_INFO,
  "pool.ntp.org",
  "time.google.com"
);
```

Le fuseau utilisé est :

```cpp
const char *TZ_INFO =
  "CET-1CEST,M3.5.0,M10.5.0/3";
```

Il correspond à la France avec passage automatique :

- heure d'hiver ;
- heure d'été.

---

# 27. Démarrage sans synchronisation NTP

Le programme tente de synchroniser l'heure pendant un temps limité.

Il effectue au maximum :

```text
20 tentatives
```

avec :

```text
500 ms
```

entre les tentatives.

Soit environ :

```text
10 secondes
```

Si la synchronisation échoue :

```text
NTP Timeout
```

le démarrage continue.

Si la box est injoignable au démarrage, cette attente est supprimée (inutile sans Internet). L'heure peut alors être réglée manuellement depuis la page Web.

###  Lecture de l'heure non bloquante

Par défaut, `getLocalTime()` attend jusqu'à **5 secondes** lorsque l'heure n'est pas encore réglée. Sans NTP, chaque appel figeait l'ESP32 (page Web qui ne répond plus, boutons ignorés).

La V3 utilise une fonction dédiée :

```cpp
bool lireHeureLocale(struct tm *t) {
  return getLocalTime(t, 10);   // 10 ms maximum
}
```

---

# 28. 🛟 Réseau Wi-Fi de secours ESP32_Secours

## Principe

L'ESP32 peut être **en même temps** client de la box (mode station) et **point d'accès Wi-Fi** pour un smartphone (mode AP).

Quand la box est injoignable depuis **30 secondes**, l'ESP32 ouvre son propre réseau :

```text
Nom du réseau (SSID) : ESP32_Secours
Mot de passe         : SECRET_AP_PASS (8 caractères minimum)
Adresse de la page   : http://192.168.4.1
```

Le serveur Web répond sur les deux réseaux à la fois : la page, les relais, les boutons et la programmation horaire fonctionnent normalement.

```text
        Box OK                              Box coupée
  ┌─────────────┐                     ┌─────────────────────┐
  │    Box      │                     │   ESP32_Secours     │
  └──────┬──────┘                     │   192.168.4.1       │
         │                            └──────────┬──────────┘
         ▼                                       │ Wi-Fi direct
   ESP32 (station)                               ▼
   richardv.local                           Smartphone
```

## Se connecter depuis un smartphone

1. Ouvrir les réglages Wi-Fi et choisir **`ESP32_Secours`**.
2. Saisir le mot de passe du réseau de secours.
3. Ouvrir le navigateur sur **`http://192.168.4.1`** (ou `http://richardv.local`).
4. Il n'est normalement **plus nécessaire de couper les données mobiles** (voir « Internet simulé » ci-dessous).
5. Si le bandeau « ⚠️ Heure non réglée » est affiché, régler l'heure (section suivante).

> 💡 Astuce : créer une fois un **raccourci sur l'écran d'accueil** du smartphone vers `http://192.168.4.1`.

## Serveur DNS et portail captif (V3.2)

Sur le réseau de secours, l'ESP32 fait aussi office de **serveur DNS** (`DNSServer`) : il répond `192.168.4.1` à **tous** les noms demandés, y compris `richardv.local`.

Toute adresse inconnue reçue par le réseau de secours est **redirigée** vers la page du programmateur.

Pourquoi : les noms en `.local` (mDNS) sont mal gérés par beaucoup de smartphones, surtout Android, sur un réseau sans Internet. L'accès « une fois sur deux » disparaît.

## Internet simulé (V3.3)

Un smartphone teste sa connexion Internet en appelant des adresses de contrôle. S'il conclut « pas d'Internet », il envoie ses requêtes par les **données mobiles** et la page devient injoignable.

Avec :

```cpp
const bool AP_INTERNET_SIMULE = true;   // réglage par défaut
```

l'ESP32 répond à ces tests **exactement comme Internet** :

| Système | Adresse testée | Réponse de l'ESP32 |
|---|---|---|
| Android / Chrome | `/generate_204`, `/gen_204` | code 204 |
| iPhone / Mac | `/hotspot-detect.html`, `/library/test/success.html` | page `Success` |
| Windows | `/connecttest.txt`, `/ncsi.txt` | `Microsoft Connect Test` / `Microsoft NCSI` |
| Firefox | `/success.txt` | `success` |

Le téléphone garde alors le Wi-Fi de secours comme connexion principale.

| `AP_INTERNET_SIMULE` | Avantage | Inconvénient |
|---|---|---|
| `true` (défaut) | Pas besoin de couper les données mobiles | La page ne s'ouvre pas toute seule : taper `192.168.4.1` |
| `false` | La page s'ouvre automatiquement à la connexion (portail captif V3.2) | Il peut falloir couper les données mobiles |

Pendant la connexion au réseau de secours, les autres applications du smartphone (mails, messageries…) n'ont pas Internet : c'est normal.

## Si le smartphone bascule encore sur les données mobiles

- **Android** : à la notification « Connectivité limitée » / « Pas d'accès Internet », choisir **« Rester connecté »** et cocher **« Ne plus demander pour ce réseau »** ;
- **Samsung** : Paramètres › Connexions › Wi-Fi › ⋮ › Intelligent Wi-Fi › désactiver **« Passer aux données mobiles »** ;
- **iPhone** : Réglages › Données cellulaires › tout en bas, désactiver **« Assistance Wi-Fi »**.

## Fermeture automatique

Le réseau de secours se referme tout seul quand :

- la box est de nouveau joignable depuis **2 minutes** ;
- **et** plus aucun smartphone n'y est connecté.

Un utilisateur en cours de réglage n'est donc jamais coupé.

## Affichage OLED pendant le secours

```text
Secours - 12:05*
AP 192.168.4.1
```

## Réseau de secours permanent (option)

Par défaut, le réseau ne s'ouvre qu'en cas de perte de la box. Pour le garder ouvert en permanence :

```cpp
const bool AP_SECOURS_TOUJOURS_ACTIF = true;
```

## Paramètres

| Constante | Valeur | Rôle |
|---|---:|---|
| `AP_SSID` | `ESP32_Secours` | Nom du réseau de secours |
| `AP_PASS` / `SECRET_AP_PASS` | 8 caractères min. | Mot de passe du réseau de secours (WPA2) |
| `AP_INTERNET_SIMULE` | `true` | Réponses « Internet OK » aux tests du smartphone |
| `AP_SECOURS_TOUJOURS_ACTIF` | `false` | `true` = réseau toujours ouvert |
| `AP_DELAI_ACTIVATION` | 30 s | Perte de box avant ouverture |
| `AP_DELAI_DESACTIVATION` | 2 min | Retour de box avant fermeture |
| `WIFI_RETRY_NORMAL` | 10 s | Recherche box sans secours |
| `WIFI_RETRY_AP_SANS_CLIENT` | 30 s | Recherche box, secours sans appareil |
| `WIFI_RETRY_AP_AVEC_CLIENT` | 3 min | Recherche box, smartphone connecté |

## Fonctions du programme

```cpp
demarrerReseauSecours();   // ouvre le point d'accès
arreterReseauSecours();    // le referme
gererReseauSecours();      // appelée chaque seconde dans loop()
dnsServer.processNextRequest(); // V3.2 : serveur DNS du réseau de secours
```

> ⚠️ **Sécurité** : toute personne à portée du Wi-Fi connaissant ce mot de passe peut commander les relais (portail, garage…). Remplacer impérativement le mot de passe par défaut.

---

# 29. 🕒 Réglage manuel de l'heure

## Pourquoi

Sans box, il n'y a ni Internet ni NTP : l'ESP32 ne connaît pas l'heure et la programmation AUTO ne peut pas fonctionner correctement.

## Depuis la page Web

1. Appuyer sur l'**horloge** en haut de la page, ou sur le bandeau jaune **« ⚠️ Heure non réglée »**.
2. Appuyer sur **« 📱 Prendre l'heure du smartphone »** (méthode recommandée), **ou** choisir une date/heure puis **« Appliquer cette date / heure »**.
3. Le message **« Heure réglée ✓ »** confirme l'opération.

Les relais en mode AUTO suivent immédiatement la nouvelle heure.

## Repères visuels

| Affichage | Signification |
|---|---|
| Horloge blanche | Heure Internet (NTP) |
| Horloge **jaune** | Heure réglée à la main |
| Horloge **rouge** + bandeau jaune | Heure inconnue |
| `12:05*` sur l'OLED | Heure réglée à la main |

La fenêtre « Infos système » indique aussi la **Source heure**.

## Retour de l'heure Internet

Dès que la box et Internet reviennent, la synchronisation NTP reprend automatiquement et remplace l'heure manuelle (notification `ntpSynchronise()` via `sntp_set_time_sync_notification_cb()`).

## Limite

L'ESP32 ne possède pas d'horloge sauvegardée par pile. Une coupure d'alimentation **pendant** une panne de box fait perdre l'heure : il faut la régler à nouveau après le redémarrage. Un module RTC (ex. DS3231) permettrait de lever cette limite.

---

# 30. Application immédiate de la programmation

Après la synchronisation NTP, le programme appelle :

```cpp
appliquerProgrammation();
```

Cela évite d'attendre le premier cycle complet de `loop()` pour corriger l'état physique des relais.

C'est particulièrement utile après :

- redémarrage ;
- coupure secteur ;
- mise à jour OTA.

---

# 31. Mode AUTO

En mode AUTO :

```text
les horaires déterminent l'état du relais
```

Toutes les secondes, le programme recalcule l'état voulu.

Si une plage est active :

```text
Relais ON
```

Sinon :

```text
Relais OFF
```

---

# 32. Mode MANUEL

En mode MANUEL :

```text
les horaires sont ignorés
```

L'état du relais est commandé par :

- bouton Web `FORCER ON/OFF` ;
- bouton physique correspondant.

Un relais qui reste en MANUEL après un redémarrage ou une OTA reste volontairement hors du contrôle horaire.

---

# 33. Retour en mode AUTO

La route :

```text
/reset-auto
```

permet de remettre tous les relais en automatique.

Cette commande :

- ne supprime pas les horaires ;
- ne réinitialise pas la NVS ;
- remet tous les relais en AUTO ;
- applique immédiatement la programmation actuelle.

Cette route ne demande pas d'authentification.

---

# 34. Sauvegarde NVS

Les réglages sont sauvegardés dans la mémoire NVS de l'ESP32 via :

```cpp
#include <Preferences.h>
```

Sont notamment conservés :

- plages horaires ;
- mode AUTO/MANUEL ;
- état mémorisé du relais.

Les réglages survivent à :

- redémarrage ;
- coupure d'alimentation ;
- mise à jour OTA.

---

# 35. Correction importante de la NVS

Dans une ancienne logique, la signature :

```cpp
__DATE__ " " __TIME__
```

pouvait être utilisée pour déterminer si la configuration devait être effacée.

Cela était problématique car cette signature change à chaque compilation.

Dans cette version :

```cpp
const char* FIRMWARE_BUILD = __DATE__ " " __TIME__;
```

sert uniquement au **diagnostic du firmware**.

La décision de réinitialiser la NVS dépend désormais de :

```cpp
const uint16_t CONFIG_VERSION = 2;
```

---

# 36. CONFIG_VERSION

Cette valeur ne doit être modifiée que si la structure des données stockées en NVS devient réellement incompatible.

Exemple :

```cpp
const uint16_t CONFIG_VERSION = 2;
```

Si une future version nécessite une nouvelle structure :

```cpp
const uint16_t CONFIG_VERSION = 3;
```

Lors du prochain démarrage, l'ancienne configuration NVS sera réinitialisée.

### Important

Ne pas augmenter cette valeur simplement parce que vous avez :

- changé le HTML ;
- changé le CSS ;
- corrigé une fonction ;
- changé un commentaire ;
- effectué une mise à jour OTA.

Sinon les réglages utilisateur seront perdus.

---

# 37. Diagnostic des mises à jour OTA

La signature :

```cpp
FIRMWARE_BUILD
```

est visible :

- dans le moniteur série ;
- sur l'OLED ;
- dans `/get-info` ;
- dans la fenêtre « Infos système ».

Cela permet de vérifier qu'un nouveau firmware est réellement exécuté.

La NVS contient également la signature du firmware précédent.

---

# 38. Mise à jour OTA

Le programme utilise :

```cpp
#include <ArduinoOTA.h>
```

Après un premier téléversement USB avec OTA actif, l'ESP32 peut être programmé par Wi-Fi.

Dans Arduino IDE :

```text
Outils
  → Port
    → Port réseau
```

puis sélectionner l'ESP32.

Le mot de passe OTA est :

```cpp
SECRET_OTA_PASSWORD
```

---

# 39. Différence entre OTA et programmation Web

Il est important de distinguer :

### OTA

Modifie :

```text
le firmware de l'ESP32
```

### Interface Web

Modifie :

```text
la configuration utilisateur
```

Une mise à jour OTA ne doit pas supprimer les horaires.

Les horaires restent dans la NVS.

---

# 40. Routes HTTP disponibles

## Page principale

```text
GET /
```

Retourne la page Web.

---

## Configuration des relais

```text
GET /get-config
```

Retourne notamment :

- nombre de relais ;
- identifiant ;
- nom ;
- sous-nom ;
- couleur ;
- GPIO ;
- nombre maximal de plages.

---

## État courant

```text
GET /get-data
```

Retourne notamment :

- plages ;
- résumé ;
- mode AUTO/MANUEL ;
- état du relais ;
- temps avant prochain changement ;
- heure courante ;
- qualité Wi-Fi ;
- `heureOK` (heure connue ou non) ;
- `heureSource` : `ntp`, `manuelle` ou `aucune`.

---

## Informations système

```text
GET /get-info
```

Retourne notamment :

- état Wi-Fi ;
- SSID ;
- hostname ;
- adresse IP ;
- adresse MAC ;
- RSSI ;
- RSSI en pourcentage ;
- signature du firmware ;
- état de réinitialisation NVS ;
- ancienne signature firmware ;
- état du réseau de secours : `apActif`, `apSsid`, `apIp`, `apClients` ;
- source de l'heure : `heureSource`.

---

## Changement AUTO/MANUEL

```text
GET /toggle-mode?id=1
```

Cette route ne demande pas d'authentification.

---

## Forçage ON/OFF

```text
GET /force-state?id=1
```

Cette route :

- passe le relais en MANUEL ;
- inverse son état ;
- applique immédiatement l'état physique ;
- sauvegarde dans la NVS.

Elle ne demande pas d'authentification.

---

## Modification des plages

```text
POST /save?id=1
```

avec :

```text
plages=06:30-08:00,18:45-22:30
```

Comme toutes les routes de cette version, elle ne demande pas d'authentification.

Exemple avec `curl` :

```bash
curl -X POST "http://richardv.local/save?id=1" \
     -d "plages=06:30-08:00,18:45-22:30"
```

Pour supprimer toutes les plages :

```bash
curl -X POST "http://richardv.local/save?id=1" \
     -d "plages="
```

---

## Retour de tous les relais en AUTO

```text
GET /reset-auto
```

Cette route ne demande pas d'authentification.

---

##  Réglage de l'heure

```text
POST /set-time
```

avec l'un des deux paramètres :

| Paramètre | Format | Usage |
|---|---|---|
| `epoch` | secondes UTC depuis 1970 | bouton « Prendre l'heure du smartphone » |
| `datetime` | `AAAA-MM-JJTHH:MM` (heure française) | saisie manuelle |

Une date antérieure au 01/01/2024 est refusée.

Exemple depuis le réseau de secours :

```bash
curl -X POST "http://192.168.4.1/set-time" \
     -d "datetime=2026-10-04T12:05"
```

---

## Adresses inconnues (réseau de secours)

Sur le réseau de secours uniquement :

- les adresses de test de connexion des smartphones reçoivent une réponse « Internet OK » si `AP_INTERNET_SIMULE = true` ;
- toute autre adresse inconnue est redirigée vers `http://192.168.4.1/`.

Sur le réseau de la box, une adresse inconnue renvoie une erreur `404` classique.

---

# 41. Anti-rebond des boutons

Le temps anti-rebond est :

```cpp
const unsigned long BP_DEBOUNCE_MS = 40;
```

Le programme distingue :

- dernière lecture brute ;
- état stabilisé ;
- instant de dernière transition.

Cela évite plusieurs déclenchements pour un seul appui mécanique.

---

# 42. Boucle principale

La fonction :

```cpp
void loop()
```

effectue notamment :

```text
1. lecture des boutons physiques
2. traitement OTA
3. surveillance Wi-Fi
4. reconnexion si nécessaire
5. recherche périodique d'un meilleur réseau
6. ouverture / fermeture du réseau de secours (gererReseauSecours)
   + réponses DNS du réseau de secours (dnsServer.processNextRequest)
7. application de la programmation horaire
8. changement éventuel de page OLED
9. mise à jour OLED
```

La programmation horaire et la surveillance Wi-Fi sont traitées environ toutes les secondes.

Les boutons et l'OTA sont traités à chaque passage de `loop()` pour conserver une bonne réactivité.

---

# 43. OLED

L'écran est organisé pour afficher notamment :

- heure ;
- réseau Wi-Fi ;
- adresse IP ;
- état des relais ;
- mode AUTO/MANUEL ;
- informations de programmation ;
- `Secours` et `AP 192.168.4.1` pendant une coupure de la box ;
- un astérisque après l'heure (`12:05*`) lorsque l'heure a été réglée à la main.

La constante :

```cpp
const int OLED_LIGNES_PAR_PAGE = 5;
```

définit le nombre de relais affichés par page.

Si le nombre de relais dépasse cette valeur, les pages tournent automatiquement.

Le changement de page intervient toutes les :

```text
8 secondes
```

---

# 44. Bibliothèques nécessaires

Le programme utilise les bibliothèques suivantes :

```cpp
WiFi.h
ESPAsyncWebServer.h
Preferences.h
ArduinoJson.h
time.h
ESPmDNS.h
DNSServer.h     //  V3.2 - fourni avec le core ESP32
ArduinoOTA.h
Wire.h
Adafruit_GFX.h
Adafruit_SSD1306.h
esp_sntp.h      //  fourni avec le core ESP32
sys/time.h      //  fourni avec le core ESP32
```

Selon la version du core ESP32 utilisée, `WiFi.h`, `Preferences.h`, `time.h`, `ESPmDNS.h`, `ArduinoOTA.h` et `Wire.h` sont fournies avec le support ESP32.

Les bibliothèques à installer si elles ne sont pas déjà présentes comprennent notamment :

- **ESPAsyncWebServer**
- **ArduinoJson**
- **Adafruit GFX Library**
- **Adafruit SSD1306**

---

# 45. Installation dans Arduino IDE

## Étape 1 – Installer le support ESP32

Dans Arduino IDE :

```text
Fichier
→ Préférences
→ URL de gestionnaire de cartes supplémentaires
```

Installer ensuite le package ESP32 approprié via :

```text
Outils
→ Type de carte
→ Gestionnaire de cartes
```

---

## Étape 2 – Choisir la carte

Sélectionner le modèle ESP32 correspondant à votre matériel.

Par exemple, pour un ESP32 DevKit classique, sélectionner la carte correspondant exactement au module utilisé.

---

## Étape 3 – Placer les fichiers

Dans le même dossier :

```text
Programmateur_horaire_ESP32_V3_Secours.ino
arduino_secrets.h
```

Le nom du dossier Arduino doit normalement correspondre au nom principal du sketch.

---

# 46. Première mise en service

Ordre recommandé :

1. vérifier le câblage ;
2. vérifier l'alimentation ;
3. vérifier les GPIO ;
4. créer `arduino_secrets.h` ;
5. installer les bibliothèques ;
6. sélectionner la bonne carte ESP32 ;
7. téléverser par USB ;
8. ouvrir le moniteur série à `115200 bauds` ;
9. vérifier la connexion Wi-Fi ;
10. vérifier l'adresse IP ;
11. vérifier le fonctionnement de `richardv.local` ;
12. vérifier l'heure NTP ;
13. vérifier chaque relais ;
14. vérifier les boutons physiques ;
15. vérifier les horaires ;
16. tester ensuite l'OTA ;
17. personnaliser `SECRET_AP_PASS` ;
18. tester le réseau de secours et le réglage manuel de l'heure (voir test ci-dessous).

---

# 47. Test recommandé des relais

Avant de connecter une charge réelle, tester les relais avec une charge de test adaptée.

Pour chaque relais :

1. ouvrir la page Web ;
2. vérifier le mode AUTO ;
3. passer en MANUEL ;
4. forcer ON ;
5. vérifier le relais ;
6. forcer OFF ;
7. vérifier le relais ;
8. revenir en AUTO ;
9. programmer une plage très courte ;
10. vérifier le changement automatique.

---

# 48. Test des plages traversant minuit

Exemple :

```text
23:55-00:05
```

Tester :

```text
23:54 → OFF
23:55 → ON
00:00 → ON
00:04 → ON
00:05 → OFF
```

Ce test permet de vérifier le fonctionnement du passage à minuit.

---

# 49. Test de conservation après redémarrage

Après avoir configuré une plage :

```text
06:32-08:10
```

faire :

1. sauvegarder ;
2. attendre la confirmation ;
3. redémarrer l'ESP32 ;
4. ouvrir la page Web ;
5. vérifier que la plage est toujours présente.

---

# 50. Test de conservation après OTA

Configurer une plage personnalisée.

Exemple :

```text
06:17-08:23
```

Puis :

1. compiler une nouvelle version ;
2. effectuer une mise à jour OTA ;
3. attendre le redémarrage ;
4. ouvrir « Infos système » ;
5. vérifier que `FIRMWARE_BUILD` a changé ;
6. vérifier que la plage `06:17-08:23` est toujours présente.

Si la plage est conservée, la gestion NVS fonctionne comme prévu.

---

# 51. Test du réseau de secours et de l'heure manuelle

1. débrancher la box ;
2. attendre 30 s : l'OLED affiche `Secours` et `AP 192.168.4.1` ;
3. connecter le smartphone à `ESP32_Secours` ;
4. **sans couper les données mobiles**, ouvrir `http://192.168.4.1` puis `http://richardv.local` et commander un relais ;
5. redémarrer l'ESP32 box débranchée : le bandeau « Heure non réglée » apparaît ;
6. appuyer sur « Prendre l'heure du smartphone » : l'horloge devient jaune, l'OLED affiche `*` ;
7. rebrancher la box et déconnecter le smartphone ;
8. après environ 2 min, le réseau de secours se ferme et l'heure NTP reprend (horloge blanche).

---

# 52. Dépannage Wi-Fi

## L'ESP32 ne se connecte pas

Vérifier :

- SSID ;
- mot de passe ;
- présence du réseau ;
- bande Wi-Fi compatible ;
- contenu de `arduino_secrets.h`.

Le moniteur série doit indiquer les tentatives de connexion.

---

# 53. `richardv.local` ne fonctionne pas

Tester d'abord l'adresse IP affichée dans le moniteur série.

Exemple :

```text
http://192.168.1.50
```

Si l'IP fonctionne mais pas :

```text
http://richardv.local
```

le problème est probablement lié à la résolution mDNS du client.

Vérifier :

- smartphone et ESP32 sur le même réseau local ;
- réseau non isolé ;
- support mDNS du navigateur/système ;
- absence de VPN bloquant le réseau local ;
- configuration du routeur.

Le fonctionnement par IP ne dépend pas de mDNS.

> 🛟 **Sans box**, connectez le smartphone au réseau `ESP32_Secours` et utilisez `http://192.168.4.1`. Depuis la V3.2, `richardv.local` y fonctionne aussi grâce au serveur DNS intégré.

---

# 54. Dépannage du réseau de secours

| Symptôme | Vérification |
|---|---|
| Un réseau `ESP_xxxxxx` (ex. `ESP_1A8241`) apparaît au lieu d'`ESP32_Secours` | C'est le point d'accès **par défaut, sans mot de passe** de l'ESP32, créé quand il refuse la configuration (mot de passe < 8 caractères, ou ouverture pendant un scan Wi-Fi). Corrigé en **V3.1** : mot de passe vérifié avant ouverture, attente de la fin des scans, contrôle du nom diffusé et fermeture immédiate de tout réseau `ESP_xxxxxx`. S'il réapparaît : vérifier `SECRET_AP_PASS` et le moniteur série (`ERREUR : SECRET_AP_PASS`, `Echec ouverture reseau de secours`, `Point d'acces inattendu`) |
| `ESP32_Secours` n'apparaît pas | Attendre 30 s après la perte de box ; vérifier que le mot de passe fait **au moins 8 caractères** (sinon erreur dans le moniteur série) |
| Connecté mais la page ne s'ouvre pas | Utiliser `http://192.168.4.1` ; vérifier `AP_INTERNET_SIMULE = true` ; appliquer les réglages smartphone (« Rester connecté », Intelligent Wi-Fi, Assistance Wi-Fi) ; couper un éventuel VPN. En dernier recours seulement, couper les données mobiles |
| `richardv.local` ne répond pas sur le secours | Vérifier que le programme est bien en V3.2 ou plus (serveur DNS) ; sinon utiliser `http://192.168.4.1` |
| La page ne s'ouvre plus toute seule à la connexion | Normal avec `AP_INTERNET_SIMULE = true` : taper `192.168.4.1` (ou mettre `false` pour retrouver l'ouverture automatique) |
| Le smartphone décroche quelques secondes | Normal : recherche périodique de la box (toutes les 3 min avec un appareil connecté) |
| Heure affichée `--:--` | Pas de NTP : régler l'heure depuis la page Web |
| Réglage de l'heure refusé | Vérifier la date du smartphone (date avant 2024 refusée) |
| Heure perdue après une coupure de courant | Normal sans module RTC : la régler à nouveau |

---

# 55. Le relais ne réagit pas

Vérifier dans l'ordre :

1. GPIO réellement utilisé ;
2. alimentation du module relais ;
3. masse commune ;
4. logique active HIGH/LOW ;
5. valeur de `relayActiveHigh` ;
6. mode AUTO/MANUEL ;
7. état de la programmation horaire.

Si le relais fonctionne à l'inverse :

```cpp
true
```

peut être remplacé par :

```cpp
false
```

pour le relais concerné.

---

# 56. Le relais reste bloqué en MANUEL

C'est normalement le comportement prévu.

Un relais en MANUEL ignore les horaires.

Utiliser :

```text
/reset-auto
```

pour repasser tous les relais en AUTO.

Cette commande ne demande pas d'authentification. Le bouton « 🔁 Tout AUTO » de la page Web produit le même résultat.

---

# 57. Une plage ne se sauvegarde pas

Vérifier :

- format `HH:MM-HH:MM` ;
- nombre de plages inférieur ou égal à `MAX_PLAGES` ;
- heure valide ;
- minutes comprises entre `00` et `59` ;
- plage non nulle.

Exemples :

```text
06:30-08:00
```

valide.

```text
06:30-06:30
```

refusé.

---

# 58. Attention aux modifications du tableau `programmateurs[]`

Les valeurs présentes dans :

```cpp
plagesDefaut
modeAuto
relayState
```

sont des **valeurs par défaut**.

Si une valeur existe déjà dans la NVS, elle est prioritaire.

Par exemple, changer :

```cpp
"06:32-08:10"
```

en :

```cpp
"07:00-09:00"
```

dans le code ne remplacera pas automatiquement une programmation déjà sauvegardée dans la NVS.

Pour modifier une programmation utilisateur, utiliser l'interface Web.

---

# 59. Ajouter un nouveau relais après utilisation

Si vous ajoutez :

```cpp
{ "5", ... }
```

le nouveau relais n'aura normalement aucune ancienne configuration NVS associée à son nouvel identifiant.

Ses valeurs `plagesDefaut`, `modeAuto` et `relayState` pourront donc servir de valeurs initiales.

---

# 60. Modifier CONFIG_VERSION

Ne faire ceci que lors d'une modification incompatible de la structure NVS :

```cpp
const uint16_t CONFIG_VERSION = 3;
```

Au prochain démarrage, la configuration du namespace `config` sera réinitialisée.

### Conséquence

Les horaires et états sauvegardés seront perdus.

Avant une telle modification, noter ou sauvegarder les horaires.

---

# 61. Structure logique du programme

Le fichier `.ino` est organisé autour des blocs suivants :

```text
Bibliothèques
    ↓
Configuration Wi-Fi
    ↓
Structures Plage / Programmateur
    ↓
Tableau programmateurs[]
    ↓
Anti-rebond boutons
    ↓
Gestion des plages
    ↓
Calcul des plages actives
    ↓
OLED
    ↓
HTML / CSS / JavaScript
    ↓
NVS / Preferences
    ↓
Wi-Fi
    ↓
mDNS
    ↓
OTA
    ↓
Setup
    ↓
Routes HTTP
    ↓
Application programmation
    ↓
Boutons physiques
    ↓
Loop
```

---

# 62. Principes importants du programme

## La NVS contient les réglages utilisateur

Elle ne doit pas être effacée à chaque OTA.

## Le tableau `programmateurs[]` contient les valeurs par défaut

Il sert à initialiser un relais qui n'a pas encore de configuration sauvegardée.

## `FIRMWARE_BUILD` sert au diagnostic

Il indique la compilation réellement installée.

## `CONFIG_VERSION` sert à la compatibilité NVS

Elle ne doit changer que si la structure NVS change.

## AUTO et MANUEL sont indépendants

MANUEL prend le contrôle du relais jusqu'au retour en AUTO.

---

# 63. Précautions électriques

Le programme peut commander des relais connectés à des tensions dangereuses.

Si les relais commandent du :

- 230 V AC ;
- moteur ;
- chauffage ;
- éclairage secteur ;
- équipement industriel ;

le câblage doit respecter les règles de sécurité électrique applicables.

L'ESP32 et sa partie basse tension doivent être correctement isolés du secteur.

Utiliser :

- boîtier adapté ;
- fusibles/protections appropriés ;
- borniers adaptés ;
- distances d'isolement suffisantes ;
- relais correctement dimensionnés.

Ne jamais manipuler un câblage secteur sous tension.

---

# 64. Résumé des paramètres principaux

| Paramètre | Valeur actuelle | Fonction |
|---|---:|---|
| `MAX_PLAGES` | 6 | Nombre max de plages/relais |
| `CONFIG_VERSION` | 2 | Version de structure NVS |
| `OLED_LIGNES_PAR_PAGE` | 5 | Relais affichés par page OLED |
| `BP_DEBOUNCE_MS` | 40 ms | Anti-rebond boutons |
| `RSSI_SWITCH_MARGIN` | 8 dBm | Marge de changement réseau |
| Nom mDNS | `richardv` | Adresse `richardv.local` |
| Port Web | 80 | HTTP |
| OLED | 128×64 | SSD1306 |
| Adresse OLED | `0x3C` | I2C |
| SDA | GPIO21 | I2C |
| SCL | GPIO22 | I2C |
| NTP | `pool.ntp.org` | Synchronisation heure |
| NTP secondaire | `time.google.com` | Synchronisation heure |
| Réseau de secours | `ESP32_Secours` | Accès sans box |
| Adresse secours | `192.168.4.1` | Page Web via le secours |
| `AP_DELAI_ACTIVATION` | 30 s | Ouverture du secours |
| `AP_DELAI_DESACTIVATION` | 2 min | Fermeture du secours |
| `AP_SECOURS_TOUJOURS_ACTIF` | `false` | Secours permanent ou non |
| `AP_INTERNET_SIMULE` | `true` | Internet simulé sur le secours |

---

# 65. Checklist avant installation définitive

- [ ] `arduino_secrets.h` correctement renseigné
- [ ] bon modèle ESP32 sélectionné
- [ ] bibliothèques installées
- [ ] OLED détecté en `0x3C`
- [ ] GPIO relais vérifiés
- [ ] logique HIGH/LOW des relais vérifiée
- [ ] alimentation correcte
- [ ] boutons correctement câblés
- [ ] Wi-Fi connecté
- [ ] adresse IP connue
- [ ] `richardv.local` testé
- [ ] heure NTP synchronisée
- [ ] toutes les plages testées
- [ ] plages traversant minuit testées
- [ ] conservation NVS testée
- [ ] OTA testée
- [ ] mode MANUEL testé
- [ ] retour AUTO testé
- [ ] charges réelles testées avec précautions
- [ ] 🆕 `SECRET_AP_PASS` personnalisé
- [ ] 🆕 réseau `ESP32_Secours` testé avec le smartphone
- [ ] 🆕 réglage manuel de l'heure testé

---

# 66. Fichiers du projet

Structure recommandée :

```text
Programmateur_horaire_ESP32_V3_Secours/
│
├── Programmateur_horaire_ESP32_V3_Secours.ino
├── arduino_secrets.h
└── README.md
```

Le fichier `arduino_secrets.h` doit rester privé car il contient les informations d'accès au Wi-Fi, le mot de passe OTA et le mot de passe du réseau de secours.

---

# 67. Version du programme documentée

Ce README correspond au programme :

```text
Programmateur_horaire_ESP32_V3_Secours.ino
```

Caractéristiques importantes de cette version :

- NVS conservée lors des compilations/OTA ordinaires ;
- version de configuration NVS ;
- diagnostic `FIRMWARE_BUILD` ;
- plages libres à la minute ;
- gestion du passage par minuit ;
- gestion de `24:00` ;
- validation avant sauvegarde ;
- aucune authentification sur l'interface Web (accès protégé par le Wi-Fi) ;
- `SECRET_OTA_PASSWORD` réservé à la mise à jour OTA ;
- gestion du niveau actif HIGH/LOW par relais ;
- application immédiate des changements ;
- Wi-Fi non bloquant ;
- mDNS ;
- OTA ;
- OLED ;
- boutons physiques ;
- réseau Wi-Fi de secours `ESP32_Secours` ;
- réglage manuel de l'heure (`/set-time`) ;
- démarrage sans box sans redémarrage en boucle ;
- lecture de l'heure non bloquante ;
- V3.1 : protection contre l'ouverture du réseau par défaut `ESP_xxxxxx` sans mot de passe ;
- V3.2 : serveur DNS et portail captif sur le réseau de secours ;
- V3.3 : Internet simulé, plus besoin de couper les données mobiles.

La structure NVS n'est pas modifiée : `CONFIG_VERSION` reste à `2` et les réglages de la V2 sont conservés lors du passage en V3.

---

## Licence / utilisation

Ce programme est destiné à un usage personnel et expérimental avec ESP32.

Avant toute utilisation avec une installation électrique ou une charge dangereuse, vérifier le dimensionnement matériel, l'isolation et les règles de sécurité applicables.

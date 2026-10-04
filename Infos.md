Ce que fait chaque ligne

      Variable              Valeur	         Rôle
AP_DELAI_ACTIVATION&emsp;&emsp;&emsp;&emsp;30 s	&emsp;&emsp;Temps d'attente entre la perte de la box et l'ouverture d'ESP32_Secours. Il évite d'ouvrir le réseau de secours pour une micro-coupure (box qui change de canal, redémarrage rapide).<br>
AP_DELAI_DESACTIVATION&emsp;&emsp;&nbsp;2 min&emsp;&emsp;Après le retour de la box, temps d'attente avant de fermer le réseau de secours, et seulement si plus aucun smartphone n'y est connecté. Il évite de fermer et rouvrir le secours si la box est encore instable.
WIFI_RETRY_NORMAL&emsp;&emsp;&emsp;&emsp;&nbsp;10 s&emsp;&emsp;Box perdue et secours pas encore ouvert : l'ESP32 recherche la box toutes les 10 s.
WIFI_RETRY_AP_SANS_CLIENT&emsp;30 s&emsp;&emsp;Secours ouvert, personne dessus : recherche de la box toutes les 30 s.
WIFI_RETRY_AP_AVEC_CLIENT&emsp;3 min&emsp;&emsp;Smartphone connecté au secours : recherche toutes les 3 min seulement. Chaque recherche coupe brièvement la liaison avec le téléphone.

Les trois lignes WIFI_RETRY_… règlent donc la recherche de la box, pas l'ouverture du secours.

Pourquoi le passage paraît si long

Le temps total est une somme :

L'ESP32 constate la perte de la box : quelques secondes, parfois jusqu'à une dizaine.
AP_DELAI_ACTIVATION : 30 s.
Une éventuelle recherche Wi-Fi en cours : 2 à 3 s. Le secours attend la fin du scan, c'est la protection contre le réseau « ESP_xxxxxx ».
Le smartphone repère le nouveau réseau : de 5 à 30 s selon le téléphone.

Au total, cela fait facilement 45 s à plus d'une minute.

Pour accélérer

Réduire le délai d'activation, par exemple à 10 s :
cpp
  const unsigned long AP_DELAI_ACTIVATION    = 10000;  // Box perdue depuis 10 s -> ouverture du réseau de secours

Je déconseille de descendre sous 5 s, pour ne pas ouvrir le secours à chaque petite coupure.

Côté smartphone : ouvrir l'écran des réglages Wi-Fi force le téléphone à rechercher les réseaux. ESP32_Secours apparaît alors beaucoup plus vite qu'en attendant.
Solution radicale : AP_SECOURS_TOUJOURS_ACTIF = true laisse le réseau de secours ouvert en permanence, donc disponible instantanément. En contrepartie, 
                    un second réseau Wi-Fi est visible en permanence, et la vérification du meilleur réseau (toutes les 60 s) perturbe brièvement 
                    un smartphone connecté au secours.

Avec 10 s (const unsigned long AP_DELAI_ACTIVATION), vous devriez passer à environ 20 à 40 s au total, dont une bonne partie dépend du smartphone.

# PGST PG-108 2G — Défaut de conception firmware : redémarrages répétés & écritures flash excessives (preuves UART)

**Langue :** [English (default)](README.md) | **Français**

## Pourquoi ce dépôt existe
J’adore cette alarme : très bonne compatibilité (Tuya/Smart Life, nombreux capteurs, etc.).  
Mais après quelques mois d’utilisation réelle, j’ai observé des **problèmes de stabilité critiques** qui la rendent peu fiable pour un usage “sécurité”.

Ce dépôt regroupe :
- symptômes vus en conditions réelles (redémarrages, voix aléatoire)
- logs UART comme preuves
- analyse technique suggérant un **défaut de conception firmware** : checks modem en boucle + **écritures trop fréquentes en mémoire non volatile** (flash/EEPROM), pouvant mener à de l’usure/corruption et à de l’instabilité.

**Appareil :** PGST Alarm Host **PG-108 2G**  
**Firmware :** `FIRMWARE: v1.25.03.n5` (vu dans les logs UART)

---

## Symptômes observés
### 1) Redémarrages répétés / instabilité
Le panneau redémarre parfois de manière répétée (dans mon cas, ça peut arriver environ toutes les heures).

**Impact sécurité :**  
Pendant un reboot, l’alarme peut être temporairement inefficace. Si un cambriolage survient à ce moment-là, c’est une vulnérabilité réelle.  
Si la sirène est en cours et que l’unité redémarre, la sirène peut s’arrêter.

### 2) Voix aléatoire en pleine nuit
Deux fois en ~4–5 mois, l’alarme s’est mise à parler toute seule en pleine nuit, sans interaction utilisateur.

---

## Preuves (log UART)
J’ai d’abord remarqué des redémarrages fréquents en surveillant le debug côté Tuya, puis j’ai approfondi avec l’UART du MCU principal.

La sortie UART montre :
- message de boot avec faute : **`hello wrold.`**
- polling modem GSM en boucle : `AT+CSQ`, `AT+CGREG?`
- `+CGREG: 0,0` (pas enregistré, par exemple pas de SIM / pas de réseau)
- lignes indiquant fortement l’appel d’une routine d’écriture flash :
  `struParaRunning Flash Program End, ...`

Un log **expurgé** d’1 minute est inclus (recommandé) :
- `evidence/log_redacted_1min.txt`

Les infos sensibles doivent être supprimées :
- MAC Wi‑Fi
- IMEI (GSN)
- `magicCode`

---

## Analyse technique (hypothèse)
Le pattern de logs suggère que le firmware :
1. boucle sur des checks modem quand l’enregistrement réseau échoue (`CGREG 0,0`)
2. déclenche très souvent des écritures en mémoire non volatile (via la répétition de `Flash Program End`)

La flash a une endurance limitée. Trop de cycles écriture/effacement peuvent provoquer :
- corruption de configuration (voix/heure/langue)
- instabilité / reboots
- panne à terme

Même sans “preuve électrique” d’usure, la répétition de “Flash Program End” pendant un polling fréquent est un gros signal de défaut de fiabilité.

---

## Correctifs firmware suggérés (côté fabricant)
- Limiter la fréquence des checks modem (backoff).
- Ne jamais écrire en flash à chaque itération.
- Écrire seulement si changement + max toutes les N minutes.
- Stockage plus robuste (journal / wear leveling).
- Garder la fonction alarme stable même si le GSM est indisponible.

---

## Appel à la communauté / prochaines étapes
J’ai d’abord remarqué des redémarrages fréquents en surveillant le debug côté Tuya. J’ai ensuite approfondi avec l’UART du MCU principal et capturé les logs présents dans ce dépôt, qui suggèrent une cause possible (polling modem en boucle + commits trop fréquents en mémoire non volatile).

**Objectif suivant :** confirmer l’hypothèse en corrélant les messages UART avec une vraie activité d’écriture flash (sniff SPI / analyse de contenu), et voir s’il existe une mitigation côté firmware (limitation de fréquence, stratégie de sauvegarde, etc.).

Si quelqu’un a :
- déjà rencontré le même problème sur **PGST PG-108 2G** (ou variantes proches),
- réussi à accéder au **SWD** / extraire des infos utiles (sans publier de binaire propriétaire),
- décodé le protocole série MCU↔Wi‑Fi (Tuya MCU) sur ce modèle,
- ou comprend la signification de `struParaRunning Flash Program End...`,

je suis preneur : ouvrez une issue / partagez vos infos.

**Note légale/sécurité :** je m’intéresse uniquement à l’analyse et à la fiabilité sur du matériel que je possède. Je ne souhaite pas publier de firmwares propriétaires ni aider à distribuer du code protégé.

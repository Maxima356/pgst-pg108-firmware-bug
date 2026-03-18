# PGST PG-108 2G — Super alarme, mais gros défaut de fiabilité firmware (reboots + corruption)

**Langue :** [English (default)](README.md) | **Français**

## Contexte
J’aime vraiment cette alarme. Niveau compatibilité elle est top (intégration Tuya/Smart Life, capteurs, etc.).  
Mais je suis tombé sur un **problème majeur de fiabilité**, au point que je ne peux pas lui faire confiance pour la sécurité.

Ce dépôt regroupe mon report détaillé avec preuves (photos + logs UART).

**Appareil :** PGST Alarm Host **PG-108 2G**  
**Alarme #1 firmware :** `v1.25.03.n5` (UART)  
**Alarme #2 (remplacement) firmware :** `v1.24.12.n5` (plus ancien, mais problème identique)

---

## Ce que j’ai observé (utilisation réelle)
### 1) Redémarrages / instabilité
J’ai d’abord vu des redémarrages via les logs debug Tuya, puis j’ai approfondi avec l’UART.

**Impact sécurité :**  
Si un cambriolage arrive pendant un reboot, l’alarme peut être inefficace.  
Et si la sirène sonne puis que l’unité redémarre, la sirène peut s’arrêter.

### 2) Le journal à l’écran montre des signes clairs de corruption
Dans les journaux (alarmes / armements), je vois des dates/heures impossibles :

- `65535-256-255 255:255:255`

Et des libellés d’événements dans des langues incohérentes ou des caractères corrompus, par exemple :
- `Péntek` (hongrois)
- `Odłączenie zasilania` (polonais)
- `Telefonado … Online` (mélangé / incohérent)
- caractères “bizarres” type `Шэ9?b?` (exemple sur photo)

Ça ressemble à des valeurs non initialisées / corrompues dans la mémoire persistante.

### 3) Voix qui parle toute seule (alarme #1)
Sur la première alarme, 2 fois en ~4–5 mois, l’alarme s’est mise à parler toute seule en pleine nuit (sans interaction).

---

## Le modèle de remplacement a aussi le problème
Après mon report, on m’a envoyé une deuxième alarme.

- Elle tourne sur un firmware **plus ancien** (`v1.24.12.n5`)
- Après seulement 1–2 semaines, je constate déjà des entrées “corrompues” dans les journaux à l’écran
- Et les redémarrages continuent (j’ai même l’impression que c’est plus fréquent)

Donc ça ne ressemble pas à un cas isolé.

---

## Preuves UART (capture 1 minute)
La sortie UART contient :
- message de boot avec faute : **`hello wrold.`**
- boucle de polling du modem GSM : `AT+CSQ`, `AT+CGREG?` avec `+CGREG: 0,0`
- message répété qui ressemble à un appel d’écriture flash :
  `struParaRunning Flash Program End...`

Un extrait **expurgé** est fourni :
- `evidence/log_redacted_1min.txt`

---

## Ce que je pense qu’il se passe (hypothèse)
Je ne peux pas le prouver à 100% sans dump/sniff de l’activité flash, mais le pattern suggère fortement :

- quand l’enregistrement GSM échoue (`CGREG 0,0`), le firmware boucle sur le polling  
- en même temps, il déclenche des écritures non volatiles de manière répétée (`Flash Program End` revient souvent)

Des écritures flash trop fréquentes peuvent expliquer :
- corruption (dates impossibles, langues mélangées, caractères bizarres)
- instabilité / redémarrages

---

## Aide de la communauté / retours d’expérience
Si vous avez :
- le même souci sur **PGST PG-108 2G** (ou variante proche),
- une explication de ce que signifie exactement `struParaRunning Flash Program End...`,
- déjà réussi à accéder au **SWD** / extraire des infos utiles sur cette carte,
- ou déjà décodé le protocole série MCU↔Wi‑Fi (Tuya MCU),

je suis preneur : ouvrez une issue ou partagez vos infos.

**Note légale :** je m’intéresse uniquement à l’analyse et à des corrections de fiabilité sur du matériel que je possède. Je ne souhaite pas publier de firmwares propriétaires ni aider à distribuer du code protégé.

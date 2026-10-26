# Câblage — SUN-2000GTIL2-LCD (GridFree G2) ↔ ESP32-C3 via le port DB-9

Objectif : lire les données de l'onduleur (dont la température interne) **sans
ouvrir l'appareil**, en remplaçant le dongle WiFi officiel par un ESP32-C3.
Le bridge parle le même protocole Modbus RTU (Eybond/Deye V100) que le dongle.

## Matériel

- ESP32-C3 (DevKitM-1 ou SuperMini) — alimentation 5 V sur la broche USB/VIN
- Module convertisseur **MAX3232** (TTL ↔ RS-232), alimenté en 3,3 V ou 5 V
- Hacheur abaisseur 12 V → 5 V (type mini360 / LM2596 breakout)
- Câble DB-9 femelle coupé, ou connecteur DB-9 femelle à vis + nappe

⚠️ Le port DB-9 de l'onduleur **n'est pas galvaniquement isolé du secteur**.
La masse du DB-9 est référencée au bus interne de l'onduleur (tension réseau
présente en amont). Consignes :

1. Flashe l'ESP32 en USB **avant** de brancher le DB-9 ;
2. Débranche l'USB, puis alimente par le 12 V de la broche 9 ;
3. Ne jamais laisser un PC relié par USB à l'ESP32 pendant que le DB-9 est branché ;
4. Boîtier de l'ESP32 isolé, rien de touchable.

## Brochage DB-9 (côté onduleur, identique à la fiche du « WiFi Plug » officiel)

| Broche DB-9 | Rôle | Vers ESP32/MAX3232 |
|---|---|---|
| 2 | RX onduleur (l'onduleur **reçoit**) | sortie **T1OUT** du MAX3232 |
| 3 | TX onduleur (l'onduleur **émet**) | entrée **R1IN** du MAX3232 |
| 5 | GND | GND commun (ESP32 GND + MAX3232 GND) |
| 9 | VCC DC 5–12 V | entrée du hacheur → **5 V → VIN** de l'ESP32 |
| 1, 4, 6, 7, 8 | NC | rien |

MAX3232 côté ESP32 : VCC → 3,3 V ou 5 V, GND → GND, **T1IN → GPIO4**, **R1OUT → GPIO5**.
(Le module a besoin de ses 4 condensateurs 100 nF/1 µF — tous les modules tout faits les ont.)

ESP32-C3 : flash et logs par son port **USB-C** ; pas de broche de sens RS-485
nécessaire (le MAX3232 est full duplex).

## Installation

1. Dans Home Assistant → module **ESPHome** → nouveau device `solar-gtil2`,
   coller le contenu de `../solar-gtil2.yaml`, brancher les secrets
   (voir `../secrets.example.yaml`).
2. **Installer/flasher d'abord l'ESP32 en USB, sans le DB-9** — vérifier que ça démarre.
3. Brancher le DB-9 (dongle WiFi **retiré** de l'onduleur — un seul maître RS-232).
4. La pince du limiteur (capteur de courant) reste sur son propre connecteur
   de l'onduleur : le bridage « injection 0 » continue de se faire en local,
   l'ESP32 ne fait que lire.

## Vérification au premier démarrage

Ouvrir les **logs** du device dans ESPHome. Le capteur de debug
« Scan registres 0-125 » logge chaque 15 s tous les registres réellement
implémentés par le firmware, ex. :

```
[gtil2_scan] registre 0 = 512 (0x0200)      <- type d'appareil
[gtil2_scan] registre 2 = 256 (0x0100)      <- protocole V1.00
[gtil2_scan] registre 70 = 2348 (0x092C)    <- 234,8 V réseau
[gtil2_scan] registre 90 = 1530 (0x05FA)    <- dissipateur : 153,0-100 = 53,0 °C
```

- **Registres 90–95 non nuls** → la température interne est disponible, les
  capteurs « Température dissipateur 1/2, transformateur, ambiante » afficheront
  des °C plausibles (30–80 °C en production, ~ambiante la nuit).
- **90–95 absents du log** → firmware qui ne les expose pas ; supprimer ces
  capteurs (ils resteri « inconnu »). Autres recours température : sonde
  externe collée/aimantée sur le dissipateur, ou écoute du bus LCD (nécessite
  d'ouvrir le capot — non retenu).

Une fois la carte validée : **supprimer le capteur debug** « Scan registres
0-125 » de la config (il est bavard dans les logs).

## Échelles de lecture (rappel)

| Donnée | Registre | Formule |
|---|---|---|
| Températures (dissipateur 1/2, inductances, transfo, amb.) | 90–95 | brut/10 − 100 |
| Tension réseau / PV | 70 / 109 | brut/10 |
| Puissances AC (86-87) / DC (82-83) | u32 | brut/10 W |
| Énergies jour / totale / temps | 60 / 63-64 / 67-68 | brut/10 |
| État | 59 | 0 veille, 1 auto-test, 2 normal, 3 warning, 4 défaut, 5 alarme |

Données du DB-9 côté GridFree : agrégées/moyennées sur ~5 min par le firmware —
le `update_interval: 15s` de la config ne rend pas les données plus fraîches,
il ne fait que les interroger souvent.

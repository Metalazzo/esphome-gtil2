# esphome-gtil2

[![CI](https://github.com/Metalazzo/esphome-gtil2/actions/workflows/ci.yaml/badge.svg)](https://github.com/Metalazzo/esphome-gtil2/actions/workflows/ci.yaml)

Bridge **ESPHome** pour onduleur GridFree **SUN-2000GTIL2-LCD (G2)** — lecture
complète des données (tension, puissances, énergies, état, **température
interne**) via le port **DB-9 « WiFi Plug »**, **sans ouvrir l'appareil** et
**sans une seule ligne de code custom**.

English: native ESPHome YAML to read a GridFree GTIL2 (G2) grid-tie inverter
over its DB-9 service port (same Modbus RTU protocol as the official WiFi
dongle), case closed, including heatsink/transformer temperatures when the
firmware exposes them.

> **Modèle concerné : SUN-1000G2 / SUN-2000GTIL2-LCD (génération 2, DC 45–90 V).**
> Les anciens SUN-1000G / SUN-2000G (**G1**) parlent la même langue mais ne
> peuplent pas les mêmes registres — voir [Limitations](#limitations).

## Pourquoi ce dépôt

Les vieilles intégrations (dont la lib `GFSunInverter` dont ce projet est le
successeur pratique) sont mortes : mécanique *custom component* supprimée
d'ESPHome, API eModbus cassée, cible G1 uniquement. Le composant officiel
`sun_gtil2` d'ESPHome est plus riche mais écoute le bus interne contrôleur→
écran, donc **démonter l'onduleur**.

Ici, on prend le chemin que le constructeur a déjà validé : le connecteur
DB-9 où se branche le dongle WiFi officiel. L'ESP32 **remplace** ce dongle et
interroge l'onduleur en Modbus RTU (esclave 1, 9600 8N1, `FC 0x03`), avec le
composant cœur `modbus_controller`. Zéro dépendance externe, validé compilé
sous **ESPHome 2026.10** (ESP32-C3, framework Arduino 3.3.12).

## Ce que ça remonte

| Donnée | Registre(s) | Échelle |
|---|---|---|
| Tension réseau (AC) | 70 | ×0,1 V |
| Fréquence réseau | 79 | ×0,01 Hz |
| Tension panneaux (PV) | 109 | ×0,1 V |
| Puissance AC injectée | 86–87 (u32) | ×0,1 W |
| Puissance panneaux (DC) | 82–83 (u32) | ×0,1 W |
| Rendement onduleur | 69 | ×0,1 % |
| Puissance limiteur (pince CT) | 99 | W |
| Énergie du jour | 60 | ×0,1 kWh |
| Énergie totale | 63–64 (u32) | ×0,1 kWh |
| Temps de production total | 67–68 (u32) | ×0,1 h |
| **Températures : dissipateur 1/2, inductance, transformateur, ambiante** | **90–95** | **brut/10 − 100 °C** |
| État onduleur (veille / auto-test / normal / warning / défaut / alarme) | 59 | — |
| Numéro de série | 3–7 | 10 octets ASCII |
| Version du protocole parlée par l'onduleur | 2 | hex (0x0100 = V1.00) |

Échelles et formules issues du protocole officiel **Eybond/Deye Modbus RTU
V100** utilisé par le firmware du G2 (voir [Sources](#sources)).

## Matériel

- ESP32-C3 (DevKitM-1, SuperMini…) — *testé en compilation ; un ESP32 classique
  ou S2/S3 fonctionne aussi en adaptant `board:` et les broches UART*
- Module convertisseur **MAX3232** (TTL ↔ RS-232, avec condensateurs intégrés)
- Hacheur abaisseur 12 V → 5 V (mini360, LM2596…) pour alimenter l'ESP32 via
  la broche 9 du DB-9 (le port fournit DC 5–12 V, comme pour le dongle d'origine)
- DB-9 femelle (câble coupé ou connecteur à vis)

⚠️ **Sécurité — le port DB-9 n'est pas galvaniquement isolé du secteur.**
Flasher l'ESP32 en USB **avant** de brancher le DB-9 ; ne jamais laisser un PC
relié par USB à l'ESP32 pendant que le DB-9 est branché ; boîtier isolé.
Détails et brochage complet : [docs/cablage.md](docs/cablage.md).

## Installation

1. **Home Assistant → module ESPHome** → *New device* → `solar-gtil2`, coller
   [`solar-gtil2.yaml`](solar-gtil2.yaml).
2. Renseigner les secrets (`wifi_ssid`, `wifi_password`, `api_key`,
   `ota_password`, `ap_password`) — voir [`secrets.example.yaml`](secrets.example.yaml)
   (le module ESPHome de HA a déjà un `secrets.yaml` global : y ajouter les clés).
3. Flasher l'ESP32 **en USB, DB-9 débranché**.
4. Retirer le dongle WiFi de l'onduleur (**un seul maître RS-232 sur le bus**),
   débrancher l'USB, brancher le DB-9. L'ESP32 démarre et s'alimente seul.
5. La **pince du limiteur reste sur son propre connecteur** : le bridage
   « injection zéro » continue de fonctionner en local, l'ESP32 ne fait que lire.

## Premier démarrage : le scan des registres

La config embarque un capteur temporaire `Scan registres 0-125 (debug)` qui
logue (tag `gtil2_scan`) **tous les registres réellement implémentés par le
firmware de votre onduleur** :

```text
[gtil2_scan] registre 0 = 512 (0x0200)     <- type d'appareil
[gtil2_scan] registre 2 = 256 (0x0100)     <- protocole V1.00
[gtil2_scan] registre 70 = 2348 (0x092C)   <- 234,8 V reseau
[gtil2_scan] registre 90 = 1530 (0x05FA)   <- dissipateur : 153,0 - 100 = 53,0 °C
```

- **90–95 présents** → la température interne est disponible : les capteurs
  afficheront des °C plausibles (30–80 °C en production, ~ambiante la nuit).
- **90–95 absents** → firmware qui ne les expose pas ; supprimer les capteurs
  température (ils restent « inconnu »). Le scan répond à la question en 2
  minutes, boîtier fermé.

Une fois la carte validée, **supprimer le capteur de scan** (il est bavard).

## Limitations

- Données du DB-9 **moyennées sur ~5 min** par le firmware GridFree (l'ancien
  cloud Eybond les affiche déjà ainsi). Pour du temps réel <2 s, il faudrait
  écouter le bus interne LCD (démontage, non couvert ici).
- Registres 90–95 : définis dans le protocole officiel V100, jamais confirmés
  publiquement sur un GTIL2 → d'où le scan intégré.
- **Lecture seule** : rien n'est écrit à l'onduleur (le bridage 0 injection
  reste piloté par le limiteur d'origine).
- G1 (SUN-1000G/2000G anciens) : la majorité de la carte fonctionne, pas la
  température. Pour un G2 avec accès interne au bus LCD, préférer le composant
  officiel [`sun_gtil2`](https://esphome.io/components/sun_gtil2.html)
  (données fraîches ~1,3 s).

## Structure

```text
solar-gtil2.yaml    config ESPHome complete (prete a coller dans HA)
secrets.example.yaml  cles a reprendre dans votre secrets.yaml
docs/cablage.md     brochage DB-9 / MAX3232 / regles d'alimentation
```

## Sources

- Protocole **Eybond Modbus RTU V100** (manuel officiel, via
  [der-kanal-homepage.de](https://www.der-kanal-homepage.de/dokumente-sun-1000-gtil))
- Manuels d'installation **SUN-xxxG2** (mêmes sources)
- Dumps réels de registres SUN-1000G : [forum MyPower.cz](https://forum.mypower.cz/viewtopic.php?t=11446)
- Specs connecteur du WiFi Plug officiel (brochage DB-9 + VCC 5–12 V)
- Docs ESPHome : [`modbus_controller`](https://esphome.io/components/modbus_controller/),
  [`sun_gtil2`](https://esphome.io/components/sun_gtil2.html)

## Licence

MIT — voir [LICENSE](LICENSE).

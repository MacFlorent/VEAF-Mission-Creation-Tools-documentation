---
description: Automatisation et surveillance des réseau IADS gérés par Skynet
---

-----------------------------

Navigation: [page "Mission Maker" du site de documentation VEAF](./index.md)

-----------------------------

🚧 **TRAVAUX EN COURS** 🚧

La documentation est en train d'être retravaillée, morceau par morceau. 
En attendant, vous pouvez consulter l'[ancienne documentation](https://github.com/VEAF/VEAF-Mission-Creation-Tools/blob/master/old_documentation/_index.md).

-----------------------------

# Table des matières

- Principes - [ici](#principes)
- Configuration - [ici](#comment-configurer-le-module)
- Exemples - [ici](#exemples-complets)

# Introduction

Le module *SkynetIadsHelper* construit automatiquement des réseaux IADS Skynet à partir des groupes présents dans la mission.  
Le module *SkynetIadsMonitor* offre des outils statiques et dynamiques pour surveiller l'état et la composition des réseaux IADS Skynet.

# Principes

[Synet-IADS](https://github.com/walder/Skynet-IADS) est un script qui pilote les systèmes radar antiaériens afin qu'ils optimisent leur survivabilité et leur léthalité en restant éteints le plus possible. Il simule ainsi un IADS (Integrated Air Defence System) dans lequel les EWR (Early Warning Radar) scannent le ciel pour repérer et surveiller des contacts, et communiquent ces informations aux sites SAM (Surface-to-Air Missile), permettant à ceux-ci de ne s'activer que lorsqu'ils sont en capacité d'engager un contact.

*Skynet-IADS* change complétement la manière dont fonctionnent les défenses antiaériennes dans DCS. Il est ainsi quasi indispensable à toute mission qui comporte des radars au sol. Malheureusement, sa configuration nécéssite de mettre en place et de maintenir un script dans chaque mission, script dont la complexité augmente avec la taille de la mission. Les modules VEAF documentés ici automatisent la construction de ce script et offrent des outils pour consulter et vérifier les réseaux ainsi créés.

# SkynetIadsHelper

Ce module construit automatiquement des réseaux IADS Skynet à partir des groupes présents dans la mission.  
Pour cela il va parcourir la liste de tous les groupes de la mission, et ajouter ceux qui sont éligibles dans les réseaux IADS Skynet. Cette initialisation des réseaux se fait au démarrage de la mission (avec un délai paramétrable) sur la base des groupes qui existent à ce moment. Les groupes en activation retardée sont intégrés, mais pas les groupes qui seront générés plus tard par script lors de la vie de la mission.

Le module va toujours créer deux réseaux Skynet : un pour la coalition *blue*, un pour la coalition *red*.

## Initialisation

Le module est activé par défaut dans les templates de mission VEAF.  
L'initialisation se fait sur le modèle suivant :

```lua
if (veafSkynet) then
    veafSkynet.PointDefenceMode = veafSkynet.PointDefenceModes.Skynet
    veafSkynet.GroupIntegrationMode = veafSkynet.GroupIntegrationModes.Strict
    veafSkynet.DynamicSpawn = false
    veafSkynet.DelayForStartup = 5
    veafSkynet.initialize(
        false, --includeRedInRadio=true
        false, --debugRed
        false, --includeBlueInRadio
        false --debugBlue
    )
end
```
### Fonction `initialize`
La fonction d'initialisation `veafSkynet.initialize` prend des paramètres qui seront passés aux réseau Skynet générés pour leur dire s'ils doivent reporter des informations.

`includeInRadio` ajoute une entrée radio F10 pour afficher ou masquer l'état du réseau.  
`debug` active l'affichage de l'état du réseau, et des logs détaillés sur le comportement de celui-ci.

### Point defences
Le module peut identifier les sites SAM capables d'intercepter des missiles antiradar, et essayer des les affecter en défense rapprochée d'autres élements du réseau à proximité.  
La propriété générale `veafSkynet.PointDefenceMode` pilote cette mécanique. Elle est à positionner avant le `initialize`. Les valeurs possibles sont décrites dans l'énumeration `veafSkynet.PointDefenceModes` :
- **`None = 0`** : le module n'essayera pas de créer des Point Defences (**valeur par défaut**)
- `Skynet = 1` : le module configure les Point Defences avec les mécaniques du script Skynet
- `Dcs = 2` : le module exclue les Point Defences du réseau IADS Skynet et les laisse aux bons soins de l'IA de DCS (toujours allumés)

Si la création de défenses rapprochée est souhaitée, le mode `Skynet` devrait être l'option à retenir. Le mode `Dcs` donne la possibilité d'avoir des défenses potentiellement plus efficaces mais aussi plus vulnérables (car toujours allumées).

### Mode d'intégration des groupes
La propriété générale `veafSkynet.GroupIntegrationMode` permet de choisir les critères que le module utilise pour décider si un groupe DCS doit être intégré dans les réseaux Skynet. Elle est à positionner avant le `initialize`. Les valeurs possibles sont décrites dans l'énumeration `veafSkynet.GroupIntegrationModes` :
- `Strict = 0` : seuls les groupes DCS composés uniquement d'unités connues de Skynet sont intégrés dans les réseaux IADS
- **`Lenient = 1`** : seuls les groupes DCS composés de au moins une unité connue de Skynet sont intégrés dans les réseaux IADS (**valeur par défaut**)

Cette propriété a un impact sur l'intégration des groupes "mixtes" dans les réseau. Typiquement, un convoi composé de tanks et transports, escorté par des SA-19, sera intégré en `Lenient` mais pas en `Strict`.

*Un groupe connu de Skynet est en règle générale un radar, un lanceur, ou une unité antiaérienne autonome (AAA, SA-15...)*

### Intégration dynamique de groupes générés en cours de mission
Les réseaux Skynet construits par `SkynetIadsHelper` sont initialisés au démarrage de la mission, sur la base des groupes existants à ce moment (qu'ils soient déjà présents ou en activation retardée).  
Le module peut aussi surveiller les groupes générés au cours du déroulement de la mission, et les intégrer dans les réseaux existants quand applicable. L'activation de cette mécanique est pilotée par la propriété générale `veafSkynet.DynamicSpawn`.
- **`false`** : le module ne surveille pas les générations de groupes en cours de mission (**valeur par défaut**)
- `true` : le module surveille les générations de groupes en cours de mission et les ajoute aux réseaux Skynet

Cette propriété peut être modifiée n'importe quand, permettant d'activer ou désactiver la surveillance des unités générées selon les besoins de la mission.

*Le module VeafSpawn ajoute les unités au sol qu'il génère dans les réseaux Skynet lorsque cela est applicable. Cette mécanique est désactivée quand `veafSkynet.DynamicSpawn` est à `true`.*

### Début de l'initialisation
L'initialisation des réseaux Skynet par le module `SkynetIadsHelper` se fait après un délai qui peut être paramétré avec la propriété générale `veafSkynet.DelayForStartup` (temps d'attend en secondes, par défaut **1 seconde**). Elle est à positionner avant le `initialize`.

## Centres de commandement et bascule des réseaux en autonome
Le module propose des outils pour ajouter et détruire des Command Centers dans les réseaux Skynet.  
Dans Skynet, un Command Center est une unité ou un statique dont dépend le fonctionnement d'un réseau. Si un réseau a au moins un Command Center, et que tous ses Command Centers sont détruits, il ne fonctionne plus et tous ses éléments basculent en autonome. Cette mécanique est intéressante pour donner des objectifs de mission, mais aussi peut donner un moyen simple de désactiver un réseau suite à un événement significatif de la mission.

Les élements autonomes restent allumés tout le temps et assurer eux-mêmes leur surveillance du ciel. Néanmoins ils bénéficient toujours des intelligences fournies par le script Skynet, en particulier ils sont toujours en capacité d'identifier un HARM et de s'éteindre pour lui échapper.

`SkynetIadsHelper` incorpore des outils pour maniupler les Command Centers des réseaux qu'il a construit.

`veafSkynet.addCommandCenterOfCoalition(iCoalitionId, sCommandCenterName)`  
Cette fonction ajoute l'élement DCS nommé en tant que Command Center pour le réseau de la coalition. L'objet DCS peut être un groupe, une unité ou un statique.

```lua
veafSkynet.addCommandCenterOfCoalition(coalition.side.RED, "CommandCenterRed")
```

`veafSkynet.destroyCommandCentersOfCoalition(iCoalitionId, iExplosionStrength)`
Cette fonction détruit (fait exploser) tous les Command Centers pour le réseau de la coalition. Attention, certains objets DCS ne sont pas destructibles par une unique explosion même de force maximale, en particulier les bunkers de commandement.

```lua
veafSkynet.destroyCommandCentersOfCoalition(coalition.side.RED)
```

*Skynet IADS propose beaucoup d'autres options pour structurer les réseaux : noeuds de communication, générateurs électriques, etc. Si le créateur de la mission souhaite exploiter ces options, il ne devrait pas passer par `SkynetIadsHelper`, mais construire lui-même son réseau manuellement.*

### Exemple d'utilisation pour une bascule en autonome d'un réseau
Dans cet exemple, on a préparé dans l'éditeur, dans un recoin inutilisé de la carte :  
- Un groupe `SkynetCommandCenterRed` qui sera utilisé comme modèle pour générer des objets Command Center. Un camion qui a peu de points de vie.
- Une zone `SkynetCommandCenterZone`

```lua
local function SkynetNetworkEnable(iCoalition)
    veaf.loggers.get(veafSkynetMonitor.Id):info("SkynetNetworkEnable " .. CoalitionString(iCoalition))
    local veafSkynetNetwork = veafSkynet.getNetwork(veafSkynet.defaultIADS[tostring(iCoalition)])
    local iads = veafSkynetNetwork.iads

	if (#iads:getCommandCenters() > 0 and iads:isCommandCenterUsable()) then
        veaf.loggers.get(veafSkynet.Id):trace("Network is already enabled")
        return
    end

    local sCommandCenterTemplateName = "SkynetCommandCenter" .. CoalitionString(iCoalition)
    local commandCenterTemplate = Group.getByName(sCommandCenterTemplateName)
    if (not commandCenterTemplate) then
        veaf.loggers.get(veafSkynet.Id):trace("Command center template group not found : " .. sCommandCenterTemplateName)
        return
    end

    local commandCenterData = mist.cloneInZone(sCommandCenterTemplateName, "SkynetCommandCenterZone")
    veafSkynet.addCommandCenterOfCoalition(iads:getCoalition(), commandCenterData.name)
    veaf.loggers.get(veafSkynet.Id):trace("Command center added [" .. commandCenterData.name .. "]")
end

local function SkynetNetworkDisable(iCoalition)
    veafSkynet.destroyCommandCentersOfCoalition(iCoalition)
end
```

## Désactivation d'un réseau
`veafSkynet.deactivateNetworkOfCoalition(iCoalitionId, elementStates)`

La désactivation d'un réseau Skynet consiste en deux étapes :
- Arrêt du script Skynet pour ce réseau.
- Bascule de tous les éléments du réseau dans un état au choix du mission maker avant de les rendre à l'IA DCS.

Les états possibles sont décrits par la propriété générale `veafSkynet.SkynetElementStates` :
- `Autonomous` : mode autonome selon configuration de chaque élément
- **`Live`** : allume tous les éléments (**valeur par défaut**)
- `Dark` : éteint tous les éléments

```lua
veafSkynet.deactivateNetworkOfCoalition(coalition.side.RED)
```

## Accéder aux réseaux IADS générés
Dans certains cas, il peut être utile au créateur de mission d'accéder aux réseaux IADS générés par `SkynetIadsHelper`. Cela peut se faire uniquement une fois que l'initialisation est terminée et doit donc être fait dans une tâche parallème qui attendra la fin de l'initialisation.

```lua
local assignRedIadsTaskId = nil
local myRedIads = nil

local function AssignRedIadsTask()
    if (not veafSkynet) then
        mist.removeFunction(assignRedIadsTaskId)
        return
    end

    if (veafSkynet.initialized) then
        mist.removeFunction(assignRedIadsTaskId)
        local veafSkynetNetwork = veafSkynet.getNetwork(veafSkynet.defaultIADS[tostring(coalition.side.RED)])
        myRedIads = veafSkynetNetwork.iads
    else
        veaf.loggers.get(veaf.Id):trace("waiting for Skynet IADS helpers to finish initialization")
    end
end

assignRedIadsTaskId = mist.scheduleFunction(AssignRedIadsTask, {}, timer.getTime() + veafSkynet.DelayForStartup + 1, 10)
```

# SkynetIadsMonitor

Le module *SkynetIadsMonitor* offre des outils statiques et dynamiques pour surveiller l'état et la composition des réseaux IADS Skynet. Il permet principalement de construire des descriptifs textuels des réseaux Skynet, afin de vérifier leurs structures.

*A compléter*

# Contacts

Si vous avez besoin d'aide, ou si vous voulez suggérer quelque chose, vous pouvez :

* contacter **Zip** sur [GitHub][Zip on Github] ou sur [Discord][Zip on Discord]
* aller consulter le [site de la VEAF][VEAF website]
* poster sur le [forum de la VEAF][VEAF forum]
* rejoindre le [Discord de la VEAF][VEAF Discord]


[Badge-Discord]: https://img.shields.io/discord/471061487662792715?label=VEAF%20Discord&style=for-the-badge
[VEAF-logo]: ../images/logo.png?raw=true
[VEAF Discord]: https://www.veaf.org/discord
[Zip on Github]: https://github.com/davidp57
[Zip on Discord]: https://discordapp.com/users/421317390807203850
[VEAF website]: https://www.veaf.org
[VEAF forum]: https://www.veaf.org/forum

[VEAF-Mission-Creation-Tools-repository]: https://github.com/VEAF/VEAF-Mission-Creation-Tools
[VEAF-mission-converter-repository]:https://github.com/VEAF/VEAF-mission-converter
[VEAF-demo-mission-repository]: https://github.com/VEAF/VEAF-Demo-Mission
[VEAF-Open-Training-Mission-repository]: https://github.com/VEAF/VEAF-Open-Training-Mission
[VEAF-Multiplayer-Missions-repository]: https://github.com/VEAF/VEAF-Multiplayer-Missions
[VEAF-Open-Training-Mission-documentation]: https://www.veaf.org/opentraining

[veaf_radio_menu]: ../images/radio_veaf_menu.png
[user_radio_menu]: ../images/radio_user_menu.png

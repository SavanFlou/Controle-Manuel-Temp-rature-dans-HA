# Création des différents éléments.

Il va falloir créer des entrées, des automatisations et des cartes.

- [Création des entrées](#création-des-entrées)
- [Création des automatisations](#Création-des-automatisations)
- [Création des fenêtres](#création-des-fenêtres)

Bien sûr, les noms que j'indique sont personnels, libre à vous d'adapter à vos envies.	

## Création des entrées
	
Prenons comme exemple que je veux contrôler mon salon.
Créons :
- une entrée logique appelée `Salon_AffichageDélai`. Elle sera utilisée pour contrôler l'apparition de la fenêtre de définition de la durée.
- une entrée numérique appelée `Salon_DélaiChauffage`. Elle sera utilisée pour savoir le temps que l'on a défini pour ce choix manuel.
			Valeur minimum = 30
			Valeur maximum = 1440
			Taille du pas = 30
			Unité de mesure = min
- une entrée numérique appelée `Salon_TempératureConsigne`. Elle sera utilisée pour conserver la température qui devrait normalement être la consigne.
			Valeur minimum = 8
			Valeur maximum = 25
			Taille du pas = 1

Les valeurs indiquées sont liées à mes choix. J'ai choisi de définir le temps de contrôle manuel par palier de demi-heure, donc je compte en min et par palier de 30, avec un maximum d'une journée. Rien ne vous empêche de faire différemment selon vos besoins. Je fonctionne en palier de 15 min pour la Salle de bains par exemple.
	
## Création des automatisations

Il faudra modifier les automatisations utilisées pour contrôler le fonctionnement de votre thermostat du salon. Personnellement, je fonctionne avec 2 automatisations, une appelée `ThermostatSalon_personne à la maison` et une `ThermostatSalon_réglage` qui règle ensuite par un système de choix. 
Le principe est donc que si le système détecte que la température demandé au thermostat est différente de la valeur de `Salon_TempératureConsigne`, il va demander via une fenêtre le temps pendant lequel on souhaite appliquer cette nouvelle température de consigne. L'automatisation `ThermostatSalon_réglage` sera désactivée, mais pas `ThermostatSalon_personne à la maison`. Ainsi si on quitte la maison, le contrôle manuel est annulé et la pièce passe en mode "veille".

1. dans les automatisations de réglage du thermostat donc dans mon cas `ThermostatSalon_réglage` et `ThermostatSalon_personne à la maison`, il faut rajouter 
- un `Quand` pour le moment où l'automatisation dans laquelle on est passe du statut désactivé au statut activé. Ainsi quand le contrôle manuel prend fin, l'automatisation se réactive et la température normale est réappliquée directement.
- une action qui détermine la température appliquée à l'entrée numérique `Salon_TempératureConsigne`.
Par exemple, une étape définit la température souhaitée dans mon salon à 20 °C, je rajoute une action. 
```yaml
action: input_number.set_value
metadata: {}
data:
  value: 20
target:
  entity_id: input_number.salon_temperatureconsigne
```
Ainsi, `Salon_TempératureConsigne` est à 20 °C tout comme le nouveau réglage de mon thermostat donc HA le détecte comme un réglage automatique.

2. `ThermostatSalon_Désactive par changement manuel`. Son but est de détecter quand on veut choisir une température manuellement et de mettre le système en route.

```yaml
alias: ThermostatSalon_Désactive par changement manuel
description: >-
  Si on change la température de consigne du thermostat dans le salon, désactive
  l'automatisation de contrôle de ce thermostat, active la fenêtre de sélection
  du temps sur l'affichage bubble
triggers:
  - trigger: template
    value_template: >-
      {{ states('input_number.salon_temperatureconsigne') | float(0) -
      (state_attr('climate.vtherm_salon', 'temperature')) | float(0) != 0}}
conditions:
  - condition: template
    value_template: >-
      {{ ( as_timestamp(now()) -
      as_timestamp(state_attr('automation.thermostatsalon_reglage',
      'last_triggered')) |int(0) ) > 3 }}
  - condition: template
    value_template: >-
      {{ ( as_timestamp(now()) -
      as_timestamp(state_attr('automation.thermostatsalon_personne_a_la_maison',
      'last_triggered')) |int(0) ) > 3 }}
actions:
  - action: input_boolean.turn_on
    metadata: {}
    data: {}
    target:
      entity_id: input_boolean.salon_affichagedelai
  - action: automation.turn_off
    metadata: {}
    data:
      stop_actions: false
    target:
      entity_id: automation.thermostatsalon_reglage
  - action: automation.turn_on
    metadata: {}
    data: {}
    target:
      entity_id: automation.thermostatsalon_reactive_controle_automatique
  - action: input_number.set_value
    metadata: {}
    data:
      value: "{{ (state_attr('climate.vtherm_salon', 'temperature'))|float(0) }}"
    target:
      entity_id: input_number.salon_temperatureconsigne
mode: single
```
Donc si la température indiquée dans la `Salon_TempératureConsigne` est différente de l'attribut `'temperature'` de `'climate.vtherm_salon'` qui est la température actuellement de consigne pour mon thermostat, cela veut dire que j'ai changé manuellement cette consigne et donc il doit falloir lancer ce système de contrôle manuel. 
Les conditions sont une "sécurité", au début j'avais mis ça en place pour être sûr que le changement de la température de consigne n'était pas dû à une automatisation, donc je vérifiais que les automatisations de contrôle de mon thermostat n'avaient pas été activées depuis moins de 3 secondes. Je crois qu'on peut s'en passer, mais ça ne me gène pas donc je les laisse.
Action: 
- active l'entrée logique qu'on a créée, pour afficher la fenêtre du choix de la durée du contrôle manuel.
- désactive mon automatisation de contrôle de mon thermostat. À vous d'adapter selon votre système.
- Active l'automatisation qui réactivera l'automatisation de contrôle automatique à la fin du temps défini.
- définit la température souhaitée comme la nouvelle température 

3. `ThermostatSalon_Réactive Contrôle Automatique`. Son but est de réactiver le contrôle automatique du thermostat à la fin du temps souhaité.

```yaml
alias: ThermostatSalon_Réactive Contrôle Automatique
description: >-
  Quand le temps défini pour le réglage manuel est fini, réactive le contrôle
  automatique du thermostat du salon
triggers:
  - trigger: template
    value_template: >-
      {{ ( as_timestamp(now()) -
      as_timestamp(state_attr('automation.thermostatsalon_desactive_par_changement_manuel',
      'last_triggered')) |int(0) ) > states('input_number.salon_delaichauffage')
      | float * 60 }}
conditions: []
actions:
  - action: automation.turn_on
    metadata: {}
    data: {}
    target:
      entity_id: automation.thermostatsalon_reglage
  - action: automation.turn_off
    metadata: {}
    data:
      stop_actions: true
    target:
      entity_id: automation.thermostatsalon_reactive_controle_automatique
mode: single
```

Donc si la durée entre maintenant et le dernier déclenchement de l'automatisation `ThermostatSalon_Désactive par changement manuel` est supérieure à l'entrée numérique `Salon_DélaiChauffage`, ça veut dire qu'on a dépassé le temps souhaité de contrôle manuel. On déclenche donc les actions pour revenir à la normale.
- activation de l'automatisation de réglage du thermostat.
- désactivation de cette automatisation.


## Création des fenêtres

Donc le principe est d'utiliser la fonction pop-up des Bubble Card, qui permet de cacher une carte jusqu'à ce qu'un évènement se passe. Dans notre cas, ce sera l'allumage de l'entrée logique appelée `Salon_AffichageDélai`.
Donc on va créer une pile verticale avec une Bubble Car en pop-up, et un numberbox pour choisir la durée.
```yaml
type: vertical-stack
cards:
  - type: vertical-stack
    cards:
      - type: custom:numberbox-card
        border: true
        entity: input_number.salon_delaichauffage
        name: Temps de réglage manuel
        icon: mdi:sofa-outline
        secondary_info: >-
          Temps pendant lequel le réglage manuel de la température sera pris en
          compte dans le Salon (+ ou - 30 minutes)
        initial: 60
        unit: " min"
        min: 30
        max: 1440
        step: 30
  - type: custom:bubble-card
    card_type: pop-up
    trigger:
      - condition: state
        entity: input_boolean.salon_affichagedelai
        state: "on"
    close_action:
      action: perform-action
      perform_action: input_boolean.turn_off
      target:
        entity_id: input_boolean.salon_affichagedelai
      data: {}
    hash: "#delaisalon"
    name: Salon
    icon: mdi:sofa
    auto_close: "15000"
```
Donc quand la condition `Salon_DélaiChauffage` est allumée, la fenêtre apparaît. Cette entrée est activée par `ThermostatSalon_Désactive par changement manuel`, et donc par un changement manuel de température de consigne. À la fermeture de la fenêtre, éteint `Salon_DélaiChauffage`. J'ai également ajouté une fermeture automatique au bout de 15 secondes, pour le cas où on changerait la température directement sur le thermostat.
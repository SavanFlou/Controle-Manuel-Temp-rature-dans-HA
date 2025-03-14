# Bonus


On peut rajouter une carte  indiquant le temps restant jusqu'au retour à la température normale liée à l'automatisation.
J'ai expérimenté 2 méthodes 
1. une carte Mushroom template.
```yaml
type: custom:mushroom-template-card
primary: |
  Contrôle manuel température
secondary: |-
  {{ ((states('input_number.salon_delaichauffage') | float * 60 -
      (as_timestamp(now()) -  as_timestamp(state_attr('automation.thermostatsalon_desactive_par_changement_manuel','last_triggered'))
      |int(0) ))/60) |round(0)}} min
icon: ""
```

2. un bouton dans une carte Bubblecard, mais dans ce cas, il faudra créer quelques autres éléments.

- une entrée numérique `Salon_TempsRestantManuel`, où on calculera le temps restant.
- un script `Salon_CalculTempsRestantManuel` 
```yaml
sequence:
  - action: input_number.set_value
    metadata: {}
    data:
      value: >-
        {{ ((states('input_number.salon_delaichauffage') | float * 60 -
        (as_timestamp(now()) -
        as_timestamp(state_attr('automation.thermostatsalon_desactive_par_changement_manuel','last_triggered'))
        |int(0) ))/60) |round(0)}}
    target:
      entity_id: input_number.salon_tempsrestantmanuel
alias: Salon_CalculTempsRestantManuel
description: Calcule le temps restant du contrôle manuel de la température du salon
icon: mdi:clock-plus
```
Donc il soustrait au Salon_DélaiChauffage la différence entre maintenant et la dernière activation du début du contrôle manuel, initié par `ThermostatSalon_Réactive Contrôle Automatique`. 
- une automatisation 
```yaml
alias: ThermostatSalon_Calcul du Temps Restant du Contrôle Manuel
description: >-
  Quand le contrôle manuel du thermostat est activé, met à jour toutes les
  minutes le temps restant pour l'afficher dans le bouton
triggers:
  - trigger: time_pattern
    minutes: /1
  - trigger: state
    entity_id:
      - input_boolean.salon_affichagedelai
    to: "off"
    from: "on"
conditions:
  - condition: state
    entity_id: automation.thermostatsalon_reactive_controle_automatique
    state: "on"
actions:
  - action: script.turn_on
    metadata: {}
    data: {}
    target:
      entity_id: script.salon_calcultempsrestantmanuel
mode: single
```
Donc toutes les minutes et aussi quand l'entrée logique est éteinte, donc quand la fenêtre de contrôle de la durée est fermée, l'automatisation lance le script pour calcule le temps restant.
Cette automatisation est par défaut désactivée, juste réactivée quand on en a besoin.

Il faut aussi rajouter 2 actions dans des automatisations déjà réglées.

- Dans `ThermostatSalon_Désactive par changement manuel`, il faut rajouter une action
```yaml
actions:
	  - action: automation.turn_on
    metadata: {}
    data: {}
    target:
      entity_id: automation.thermostatsalon_calcul_du_temps_restant_du_controle_manuel
```	  
Pour activer ce calcul au début.

Dans `ThermostatSalon_Réactive Contrôle Automatique`
```yaml
  - action: automation.turn_off
    metadata: {}
    data:
      stop_actions: true
    target:
      entity_id: automation.thermostatsalon_calcul_du_temps_restant_du_controle_manuel
```
Pour désactiver cette automatisation devenue inutile quand l'automatisation du thermostat redémarre.
À noter qu'il faut bien laisser l'action de désactivation de `ThermostatSalon_Réactive Contrôle Automatique` en dernier. Sinon l'automatisation est désactivée et ne lit pas les étapes d'après.

Ensuite dans la carte bubblecard, il faut rajouter dans la section `sub_button`,
```yaml
  - entity: input_number.salon_tempsrestantmanuel
    show_icon: false
    show_name: false
    show_state: true
    name: Temps Restant Manuel
    tap_action:
      action: none
    visibility:
      - condition: state
        entity: automation.thermostatsalon_reactive_controle_automatique
        state: "on"
``` 
Le temps restant s'affichera dans le bouton, qui sera visible que si l'automatisation `ThermostatSalon_Réactive Contrôle Automatique` est activé donc si le contrôle manuel est en cours.




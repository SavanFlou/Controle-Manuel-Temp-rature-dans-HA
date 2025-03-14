# Présentation

- [Présentation](#présentation)
- [Principe](#principe)
- [Configuration](#configuration)


Si vous avez automatisé le contrôle de vos thermostats, il arrive que vous souhaitiez modifier la température pour X raisons. Mais suivant vos automatisations, ce contrôle manuel peut être assez aléatoire.
Pour éviter, j'ai créé cette méthode permettant de désactiver les automatisations pendant un temps que l'on choisit.

## Principe
	
Le principe est simple :
- Vous modifiez manuellement la température.
- une fenêtre apparaît, proposant de choisir le temps pendant lequel cette nouvelle température restera la température de consigne. Après ce temps, la température normale définie par l'automatisation sera réappliquée.
	
## Configuration
	
Ma configuration dans Home assistant est la suivante 
- Pour le contrôle et la visualisation, j'utilise Bubble Card. Ce n'est pas nécessaire pour le contrôle de la température mais ça le sera pour afficher les fenêtres de choix de la durée.
- pour les automatisations, j'ai 2 automatisations par pièce, une pour mettre la pièce  en "sommeil" quand personne n'est à la maison, et une pour contrôler la température en fonction des moments de la semaine et de la journée.
	
	
	

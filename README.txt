Exploration de données NOAA


Projet perso pour apprendre à manipuler des données réelles de pertes 
catastrophe naturelle via Python

Ce que j'ai fait dans ce premier notebook

J'ai pris des données réelles NOAA (ncei.noaa.gov) (événements météo USA 2023, 
~75k événements), filtré uniquement la grêle (~11.7k), puis isolé 
les événements avec des pertes assurées déclarées (850 au final).

Les montants de pertes étaient en texte ("100.00K", "1.00M"), 
j'ai écrit une fonction pour les convertir en vrais nombres.

En traçant la distribution dans un histogramme, on voit clairement que la plupart des 
événements ont des pertes faibles, mais quelques-uns sont énormes 
(plusieurs millions), ce qui donne une distribution asymétrique à droite. 
C'est exactement le genre de comportement qu'on attend des risques 
cat nat (peu d'événements concentrent l'essentiel des pertes).

Plus de la moitié des événements grêle n'ont aucun dégât déclaré. Soit parce qu'il n'y 
a vraiment rien eu, soit parce que ça n'a pas été reporté.

## Stack
Python, pandas, numpy, matplotlib

## Et après ?
Maintenant que je sais manipuler les données, j'ai envie d'aller plus 
loin et d'essayer de fitter une vraie loi statistique sur ces pertes 
(Pareto/Weibull) pour voir si ça colle avec ce que j'observe.



2. Loss Distribution Modeling

A partir de là je voulais modéliser mathématiquement pour pouvoir répondre à des vraies questions actuarielles : 
quelle perte ne sera dépassée que 1% du temps ? Combien un assureur devrait-il provisionner par an ?

Avant de choisir une loi au hasard, j'ai tracé un Mean Excess Plot qui révèle la structure de la queue en calculant, pour chaque seuil u, de combien les pertes le dépassent en moyenne. La courbe forme une cloche (monte puis redescend), ce qui suggère une lognormale plutôt qu'une Pareto pure.

J'ai ensuite fitté trois lois sur les 850 événements. La loi lognormale, la loi de Pareto généralisée et la loi de Weibull. Ensuite j'ai comparé leur qualité via l'AIC et un QQ-plot. La lognormale gagne (AIC le plus bas), mais le QQ-plot révèle qu'elle sous-estime les événements extrêmes. J'ai donc gardé la Pareto comme scénario de stress pour le capital.

Un truc intéressant : la Pareto fittée a un shape de 1.98, ce qui signifie que sa moyenne théorique est mathématiquement infinie. 
Concrètement, ça veut dire que plus on accumule de données, plus la moyenne empirique continue de grimper. Le risque n'est donc pas bornables au sens classique. C'est  le type de risque que les cat bonds sont conçus à absorber.

Pour la fréquence, j'ai utilisé une loi de Poisson (λ=850 événements/an) 
et calculé une prime pure = fréquence × sévérité moyenne. La prime théorique (714M) est 3x inférieure à la prime empirique (2.2Mrd). 
Ce n'est pas parce que le modèle est faux, mais parce qu'une seule année de données avec une queue aussi épaisse est trop instable pour calibrer 
correctement. C'est la vraie conclusion de ce niveau : il faut plus de données.

## Et après ?
Le niveau 3 va télécharger plusieurs années (2015-2023) pour construire 
une vraie série temporelle et voir si la distribution des pertes 
évolue avec le temps.

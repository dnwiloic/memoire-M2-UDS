# Chapitre — Contribution

> Document de rédaction. Les sources mobilisées sont référencées en notes entre
> crochets `[...]` et récapitulées en fin de chapitre.

---

## 1. Introduction

La contamination des eaux souterraines par les substances per- et polyfluoroalkylées
(PFAS) est devenue un problème de santé publique de premier plan. Ces composés, employés
depuis les années 1950 dans l'industrie et les mousses anti-incendie, sont à la fois
persistants, mobiles dans les aquifères et toxiques à de très faibles concentrations.
En 2024, l'agence américaine de protection de l'environnement a fixé pour la première fois
des limites maximales réglementaires (NPDWR), définissant un seuil de dépassement par
molécule ainsi qu'un indice de risque pour les mélanges [EPA 2024 NPDWR]. Disposer d'un
moyen de prédire où ces seuils sont susceptibles d'être franchis, sans avoir à analyser
chaque forage, présente donc un intérêt opérationnel direct pour la priorisation de la
surveillance.

La Californie offre un terrain d'étude privilégié pour cette question : son réseau de
suivi des eaux souterraines est dense, et un jeu de données récemment publié rassemble
des dizaines de milliers de mesures de PFAS sur l'ensemble de l'État [Dong et al. 2024].
Ce jeu de données constitue le support empirique de l'ensemble de la contribution
présentée ici. Sa caractéristique essentielle est que la simple mesure ponctuelle d'un
forage ne suffit pas à expliquer la contamination : celle-ci dépend du contexte —
proximité de sources industrielles ou militaires, nature du sol, hydrogéologie locale,
co-polluants présents. Exploiter ce contexte suppose d'abord de l'attacher aux
observations, c'est-à-dire d'enrichir les données brutes par des variables
environnementales géoréférencées.

Or l'enrichissement géospatial est, dans la plupart des travaux, un assemblage de scripts
ad hoc, étroitement liés à un jeu de données particulier et difficilement réutilisables.
Le code initial de ce projet ne faisait pas exception : la logique de jointure tenait dans
un module monolithique de près de mille lignes, où la mécanique générique d'appariement
spatio-temporel était indissociable des spécificités du dataset PFAS californien
[RAPPORT_OUTIL.md]. Cette situation soulève un premier verrou, d'ordre méthodologique et
logiciel : comment construire un outil d'enrichissement qui sépare proprement ce qui
relève de la donnée étudiée de ce qui relève du contexte géographique, de façon à pouvoir
le réutiliser sur d'autres polluants ou d'autres régions ?

Un second verrou, scientifique celui-là, tient à l'évaluation des modèles de prédiction.
La contamination PFAS est fortement auto-corrélée dans l'espace : deux forages voisins se
ressemblent. Un modèle qui dispose de la localisation peut alors atteindre des
performances spectaculaires en mémorisant la géographie plutôt qu'en apprenant des
mécanismes, et cette mémorisation s'effondre dès qu'on lui demande de prédire dans une
zone qu'il n'a jamais vue. La plupart des scores rapportés dans la littérature, obtenus
par découpage aléatoire des données, surestiment ainsi la capacité réelle de
généralisation. Évaluer honnêtement un modèle géospatial impose au contraire de tester sa
généralisation hors-zone, par une validation croisée spatiale qui isole des blocs
géographiques entiers.

La contribution de ce chapitre s'organise autour de ces deux verrous. Le premier volet est
un outil, `geoenrich`, moteur d'enrichissement géospatial générique fondé sur un contrat
d'entrée minimal et une bibliothèque d'opérateurs enfichables ; il a été conçu pour être
indépendant du jeu de données PFAS qui lui a servi de premier banc d'essai, et cette
indépendance est étayée par une validation de parité avec le pipeline d'origine
[RAPPORT_OUTIL.md ; PARITY.md]. Le second volet est un banc de cinq modèles de prédiction
du risque de dépassement réglementaire — deux modèles tabulaires de référence, un
transformeur de graphe hétérogène, et deux modèles hybrides — comparés à protocole identique.
En réponse au second verrou, la localisation est retirée des variables de tous les modèles et
ne subsiste que dans la topologie d'un graphe relationnel : on teste ainsi si le signal
spatial doit être mémorisé comme variable ou structuré dans le graphe. La généralisation
hors-zone, qu'une validation croisée spatiale par blocs permettrait de mesurer, est discutée
à ce titre.
L'interprétabilité, globale et locale, accompagne l'ensemble afin que les prédictions
restent lisibles pour un usage environnemental. Le chapitre se clôt sur une synthèse des
apports et des limites. Il est organisé comme suit : la section 2 présente la vue
d'ensemble de l'approche ; la section 3 décrit l'outil `geoenrich` ; la section 4 expose
les modèles de prédiction ; la section 5 rassemble les résultats, leur interprétabilité et
leur discussion ; la section 6 conclut.

![Schéma 1 — Pipeline global](render/latex/schema1.png)

***Schéma 1.*** *Vue d'ensemble du pipeline, de l'observation ponctuelle à l'évaluation
interprétée.*

---

## 2. Vue d'ensemble de l'approche

Avant d'entrer dans le détail de l'outil d'enrichissement puis des modèles, cette section
décrit la chaîne de traitement dans son ensemble et les principes qui en gouvernent la
conception. L'objectif est de donner une lecture de bout en bout, de l'observation brute
jusqu'à la prédiction interprétée, et de situer chaque contribution dans ce fil.

### 2.1 Chaîne de traitement de bout en bout

Le pipeline s'articule en quatre étapes successives. En entrée, des observations
ponctuelles : pour chaque prélèvement, un identifiant, une position géographique et une
date. Ces observations passent d'abord par l'étape d'**enrichissement**, qui leur attache
des dizaines de variables contextuelles décrivant l'environnement du point — sols,
hydrologie, sources de contamination, co-polluants, unités hydrogéologiques. Le jeu
enrichi alimente ensuite une étape de **prétraitement** (définition de la cible
réglementaire, sélection et nettoyage des variables, séparation des données), puis l'étape
de **modélisation** qui entraîne et compare plusieurs prédicteurs. La dernière étape,
**évaluation et interprétation**, mesure les performances, rend les décisions des modèles
lisibles et restitue les prédictions sous forme cartographique.

![Schéma 1bis — Pipeline détaillé](render/latex/schema1bis.png)

***Schéma 1bis.*** *Architecture interne des étapes : enrichissement (`geoenrich`),
prétraitement (cible EPA 2024, anti-fuite, découpage) et modélisation (banc de cinq modèles).*

Cette linéarité apparente masque une distinction structurante, introduite dès la première
étape et reprise dans tout le reste du travail : la séparation entre la *charge utile* —
ce que l'on mesure et cherche à prédire — et le *contexte géographique* — où et quand la
mesure a été faite. C'est cette distinction qui fait l'objet de la section suivante.

### 2.2 Du jeu d'observations au dataset enrichi

L'enrichissement repose sur une idée directrice : tout le contexte géographique d'une
observation se déduit de son *où* et de son *quand*, indépendamment de la nature de ce qui
est mesuré [README.md ; RAPPORT_OUTIL.md]. Un opérateur qui calcule la distance au site de
contamination le plus proche, ou qui rattache un point à son sous-bassin hydrogéologique,
n'a pas besoin de savoir s'il manipule une mesure de PFAS, de nitrate ou de métaux : il
n'utilise que les coordonnées et la date. La charge utile est transportée sans jamais être
lue.

Cette séparation a une conséquence pratique forte. Elle permet de réduire le contrat
d'entrée à quatre champs canoniques — identifiant, latitude, longitude, date — auxquels
n'importe quel jeu d'observations peut être ramené, le reste des colonnes étant considéré
comme charge utile ignorée [README.md]. L'enrichissement devient alors une opération
générique, applicable au-delà du cas PFAS qui lui a servi d'instanciation. C'est le
fondement de l'outil `geoenrich` détaillé en section 3.

Sur le dataset californien, cette étape attache aux mesures un large éventail de variables
de contexte : proximité et densité de sites GeoTracker, de stations d'épuration, de sites
fédéraux liés à la défense, de décharges ; propriétés des sols ; variables
hydrogéologiques ; sous-bassins SGMA [california.yaml]. La validité de cet enrichissement
a été contrôlée par comparaison avec le pipeline initial du projet : sur les 94 colonnes
d'enrichissement, 77 sont identiques, 17 sont mieux renseignées et aucune ne régresse
[PARITY.md ; RAPPORT_OUTIL.md].

### 2.3 De l'enrichissement à la prédiction

Le dataset enrichi sert de socle commun à tous les modèles, ce qui garantit une comparaison
équitable. Sur ce socle, deux familles de modèles cohabitent. Les modèles **tabulaires**
(forêt aléatoire, gradient boosting) traitent chaque observation comme un vecteur de
variables indépendant des autres. Le modèle **relationnel** (transformeur de graphe
hétérogène) exploite au contraire la structure : il relie les forages entre eux et à des
entités partagées — sites de contamination, sous-bassins, voisinages spatiaux — de sorte
que des points proches d'une même source échangent de l'information [notebook 10].
Deux modèles **hybrides** complètent le banc, en combinant les représentations apprises par
le graphe avec les variables tabulaires.

Cette cohabitation n'est pas une simple juxtaposition : elle pose une question de recherche
précise. Le signal spatial, qui est manifestement informatif, doit-il être *mémorisé* en
tant que variable d'entrée, ou *structuré* dans la topologie d'un graphe ? La conception du
banc permet de trancher empiriquement, en retirant la localisation des variables d'entrée et
en ne la laissant intervenir que par la structure du graphe relationnel [notebook 10].

### 2.4 Principes directeurs

Trois principes traversent l'ensemble de la contribution et en assurent la cohérence.

**Réutilisabilité de l'outil.** L'enrichissement est conçu comme un composant générique et
non comme un script jetable. Le moteur ne contient aucune référence au cas PFAS ; les
spécificités régionales sont reléguées dans une configuration externe et un mapping
d'entrée, tous deux remplaçables sans toucher au code [RAPPORT_OUTIL.md].

**Rigueur de l'évaluation.** Le dispositif central contre la mémorisation géographique est
le retrait de la localisation des variables d'entrée, combiné à sa structuration dans la
topologie du graphe relationnel. Une validation croisée spatiale par blocs, qui mesure la
généralisation à des zones non vues, complète ce dispositif comme diagnostic ; sa portée et
son statut dans ce travail sont discutés en section 5.

**Interprétabilité comme exigence transversale.** Chaque modèle est accompagné d'analyses
d'importance des variables et d'explications locales, et le modèle de graphe fait l'objet
d'une lecture pas à pas de son fonctionnement. Dans un contexte environnemental où les
prédictions peuvent orienter des décisions de surveillance, la lisibilité des modèles est
traitée comme un livrable à part entière, et non comme un complément optionnel.

---

## 3. `geoenrich` : un moteur d'enrichissement géospatial générique

La première contribution de ce travail est un outil. `geoenrich` est un moteur
d'enrichissement géospatial qui, à partir d'un jeu d'observations ponctuelles respectant
un contrat d'entrée minimal, attache des dizaines de variables contextuelles via une
bibliothèque d'opérateurs enfichables [RAPPORT_OUTIL.md ; README.md]. Le dataset PFAS
californien en est la première instanciation et le banc de validation, non une dépendance.
Cette section expose la motivation de l'outil, son contrat d'entrée, son architecture en
trois couches, sa bibliothèque d'opérateurs, ses entrées et sorties, puis la validation de
sa correction et de sa généricité.

### 3.1 Motivation

L'enrichissement géospatial est, dans la pratique courante, un travail de scripts ad hoc.
Le code initial du projet illustrait ce travers : la logique de jointure tenait dans un
module unique, `merge.py`, de près de mille lignes, où la mécanique générique
d'appariement spatial et temporel se trouvait inextricablement mêlée aux particularités du
cas PFAS — noms de colonnes propres au programme GAMA, chemins de couches spécifiques,
règles métier liées à la nature du polluant [RAPPORT_OUTIL.md]. Un tel code n'est ni
testable isolément, ni réutilisable sur un autre jeu de données : changer de polluant ou
de région imposerait de le réécrire.

L'objectif de `geoenrich` est de transformer cette logique monolithique en un composant
réutilisable. Le principe retenu pour y parvenir est une séparation stricte entre le *quoi*
— la mesure étudiée, qui n'est qu'une charge utile transportée — et le *où/quand* — les
clés de jointure universelles que sont la position et la date. Puisque tout l'enrichissement
ne dépend que du où/quand, il peut être rendu indépendant de la nature des données
[RAPPORT_OUTIL.md].

### 3.2 Contrat d'entrée

Le contrat d'entrée est l'interface minimale que tout jeu d'observations doit satisfaire
pour être enrichi. Il se réduit à quatre champs canoniques : un identifiant de point
(`entity_id`), une latitude et une longitude en WGS84 (les clés de jointure spatiale), et
une date (clé de jointure temporelle). Seules la latitude et la longitude sont strictement
obligatoires ; l'identifiant est généré à défaut, et la date n'est requise que si des
opérateurs temporels sont mobilisés [README.md]. Toute autre colonne du jeu d'entrée est
considérée comme charge utile et n'est jamais lue par le moteur.

Le passage au contrat est assuré par une fonction `to_contract(df, mapping)` qui renomme
les colonnes du jeu source vers ces noms canoniques [README.md]. Une étape de validation
contrôle ensuite le typage, supprime les lignes sans coordonnées et analyse les dates.
C'est cette réduction à quatre champs qui rend le moteur agnostique : ce qu'il manipule
n'a, de son point de vue, aucune nature particulière.

### 3.3 Architecture en trois couches

L'architecture de `geoenrich` repose sur trois couches strictement séparées, dont la
généricité décroît à mesure que l'on descend vers le cas concret [RAPPORT_OUTIL.md].

![Schéma 2 — geoenrich en trois couches](render/latex/schema2.png)

***Schéma 2.*** *Les trois couches de `geoenrich` : contrat agnostique, opérateurs
paramétrés (primitives spatiale et temporelle), configuration spécifique à la région.
(d'après `RAPPORT_OUTIL.md`)*

#### 3.3.1 Couche contrat — générique et agnostique

La couche supérieure est celle du contrat décrit ci-dessus. Entièrement agnostique, elle
ne connaît que les quatre clés canoniques et ignore tout de la charge utile. C'est le point
d'entrée commun par lequel n'importe quel jeu d'observations est normalisé avant traitement.

#### 3.3.2 Couche opérateurs — primitives de jointure

La couche intermédiaire rassemble les opérateurs d'enrichissement, eux aussi génériques
mais paramétrables. Ils reposent sur deux primitives de jointure seulement. La primitive
**spatiale** s'appuie sur un arbre k-d en distance grand-cercle, qui permet de retrouver
efficacement le point source le plus proche, les sources dans un rayon donné, ou le
polygone contenant un point. La primitive **temporelle** repose sur un appariement par
antériorité (`merge_asof`) par entité, avec repli spatial lorsque l'appariement direct
échoue [README.md ; RAPPORT_OUTIL.md]. Chaque opérateur expose la même signature
`apply(df) -> df`, n'ajoute que des colonnes, et reste indépendant de la charge utile.

#### 3.3.3 Couche configuration — spécifique à une étude

La couche inférieure est la seule à porter le contexte particulier d'une région ou d'une
étude. Une configuration au format YAML déclare la racine des données de référence et la
liste ordonnée des opérateurs à appliquer, chacun pointant vers ses couches locales. C'est
ici, et ici seulement, que se concentrent les choix propres au cas californien
[california.yaml]. Un registre interne associe chaque type d'opérateur déclaré dans la
configuration à sa classe d'implémentation, de sorte que l'ajout d'un opérateur à un
enrichissement ne demande qu'une ligne de configuration, sans modification du code.

### 3.4 Bibliothèque d'opérateurs

Les opérateurs constituent la boîte à outils de l'enrichissement. La bibliothèque en
compte cinq familles documentées, chacune répondant à un motif de jointure récurrent
[README.md] :

- **`polygon_join`** rattache un point aux attributs du polygone qui le contient (par
  exemple le sous-bassin hydrogéologique d'un forage) ;
- **`proximity`** calcule la distance au site le plus proche, son type, et le nombre de
  sites présents dans une série de rayons ;
- **`nearest_point`** récupère les valeurs du point source le plus proche, dans une version
  statique ou indexée par année et par mois ;
- **`nearest_by_group`** traite les sources au format long en retrouvant, pour chaque code
  de variable, le moniteur le plus proche ;
- **`temporal_asof`** réalise l'appariement temporel par entité avec repli spatial.

Ces opérateurs partagent quatre propriétés qui font la robustesse du moteur : ils sont
**composables** (un enrichissement est une succession d'opérateurs), **optionnels** (un
opérateur dont la source est absente est ignoré sans interrompre le traitement),
**idempotents** sur le contrat (ils n'ajoutent que des colonnes, sans jamais altérer
l'entrée), et **indépendants de la charge utile** [RAPPORT_OUTIL.md].

### 3.5 Entrées et sorties

En entrée, `geoenrich` accepte des observations ponctuelles au format CSV, Parquet ou
GeoJSON, ramenées au contrat par leur mapping. La sortie est le même jeu, augmenté des
colonnes produites par chaque opérateur, accompagné d'un dictionnaire de données décrivant
les variables générées [README.md]. L'outil est utilisable à la fois comme bibliothèque
Python et en ligne de commande, cette dernière prenant en arguments le jeu d'entrée, le
mapping, la configuration et le fichier de sortie [README.md].

Pour le cas californien, la configuration `california.yaml` instancie neuf opérateurs qui
mobilisent autant de couches de référence [california.yaml] : sous-bassins hydrogéologiques
SGMA ; sites de contamination GeoTracker ; stations d'épuration (EPA FRS/ICIS-NPDES) ;
sites fédéraux liés à la défense (source majeure de PFAS par les mousses anti-incendie) ;
décharges (CalRecycle SWIS) ; propriétés des sols SSURGO ; variables hydrologiques
mensuelles NASA GLDAS ; qualité de l'air EPA AQS ; et co-contaminants issus du programme
GAMA par appariement temporel. À ces opérateurs s'ajoutent l'occupation du sol (NLCD) et la
topographie (USGS 3DEP) prélevées par échantillonnage raster. L'ensemble produit, à partir
des seules coordonnées et dates, un contexte environnemental riche pour chaque mesure.

### 3.6 Validation par parité

Un moteur réécrit à partir d'un code monolithique doit prouver qu'il reproduit le
comportement d'origine. Cette validation a été menée par un test de parité : faire passer
la base PFAS par le moteur générique, configuré par `california.yaml`, et comparer les
colonnes produites à celles du pipeline initial. Sur les 94 colonnes d'enrichissement, 77
sont strictement identiques, 17 (relatives aux sols) sont mieux renseignées que dans
l'original, et aucune ne régresse [PARITY.md ; RAPPORT_OUTIL.md]. Ce résultat a été obtenu
sans écrire la moindre ligne de code spécifique au cas PFAS : la généricité du moteur n'a
pas été payée au prix d'une perte de fidélité. Le moteur est par ailleurs couvert par une
quinzaine de tests unitaires sur données synthétiques [RAPPORT_OUTIL.md].

### 3.7 Généricité et indépendance

L'indépendance du moteur vis-à-vis du cas PFAS, et plus précisément du programme GAMA dont
sont issues les données, mérite d'être établie avec précision car c'est l'argument central
de la réutilisabilité. Elle repose sur plusieurs constats [RAPPORT_OUTIL.md].

D'abord, le code du moteur — contrat, primitives, opérateurs, orchestration, interface en
ligne de commande — ne contient aucune référence fonctionnelle à GAMA ni à PFAS : ni nom de
colonne, ni chemin, ni règle métier propre au polluant. Les rares occurrences relevées sont
des commentaires de documentation ou un préfixe par défaut purement cosmétique, surchargé
par la configuration.

Ensuite, le couplage au cas californien est entièrement reporté dans deux fichiers externes
et remplaçables : l'exemple d'entrée, qui fournit les forages GAMA via leur mapping, et la
configuration, qui pointe vers les couches de référence locales. Changer de jeu de données
ou de région ne demande qu'un nouveau mapping et une nouvelle configuration, sans toucher
au moteur.

Une nuance d'honnêteté doit cependant être posée. L'opérateur d'appariement temporel des
co-contaminants effectue une jointure par entité entre l'entrée et la source. Lorsque les
deux partagent le même espace d'identifiants — cas PFAS, où entrée et co-contaminants
viennent de GAMA — l'appariement direct fonctionne ; pour un jeu de données tiers dont les
identifiants ne correspondent pas, l'opérateur bascule automatiquement sur le repli
spatial. Il ne s'agit donc pas d'une dépendance de code mais du comportement générique
correct, qui était d'ailleurs déjà majoritaire sur le cas PFAS lui-même (l'appariement
direct ne concernait que 896 lignes sur 46 338, le reste passant par le repli spatial)
[RAPPORT_OUTIL.md]. Un jeu tiers se comporterait ainsi comme la majorité des lignes
actuelles.

L'indépendance est donc établie par inspection du code et par conception. La preuve
empirique décisive — faire passer par l'outil un jeu d'observations d'un autre polluant ou
d'une autre région et constater qu'il en ressort enrichi — reste à exécuter ; c'est le test
de généralité qui transformerait une indépendance « par construction » en une indépendance
« démontrée » [RAPPORT_OUTIL.md].

### 3.8 Pertinence et positionnement

L'apport de `geoenrich` se mesure au regard des pratiques courantes d'enrichissement. Là où
celles-ci produisent des scripts liés à un jeu de données et difficilement transférables,
l'outil propose une séparation nette entre une mécanique générique, validée et testée, et
des choix régionaux confinés à une configuration déclarative. Cette organisation rend
l'enrichissement reproductible, auditable opérateur par opérateur, et extensible sans
réécriture. Au-delà du présent travail sur les PFAS, elle ouvre la voie à une réutilisation
sur d'autres contaminants ou d'autres territoires, ce qui constitue la contribution
logicielle de ce mémoire.

---

## 4. Modèles de prédiction de la contamination

La seconde contribution est un banc de cinq modèles prédisant, pour chaque prélèvement, le
risque de dépassement du seuil réglementaire EPA 2024. Tous partagent le même dataset
enrichi, la même cible et le même découpage, ce qui rend leur comparaison équitable. Le fil
directeur de cette section est une question de recherche : le signal spatial, qui est
fortement informatif, doit-il être *mémorisé* comme variable d'entrée ou *structuré* dans la
topologie d'un graphe ? Pour y répondre, la localisation est retirée des variables de tous
les modèles et n'intervient que dans la construction du graphe relationnel
[notebook 10].

> Les figures référencées dans cette section et la suivante proviennent du livrable
> `ca-pfas-ml/livrables/10/PFAS_HGT_relationnel_outputs_20260627_1230/`. Chaque appel de
> figure indique le **nom de fichier** à insérer. Les schémas d'architecture (Schéma 3 à 5)
> sont, eux, produits pour ce mémoire.

![Schéma 3 — Banc des cinq modèles](render/latex/schema3.png)

***Schéma 3.*** *Le banc des cinq modèles sur socle commun (dataset enrichi, découpage
60/20/20) et les dépendances des modèles hybrides (Embedding Fusion, Stacking).*

### 4.1 Données et cible

#### 4.1.1 Le dataset enrichi CA-PFAS

L'étude porte sur 46 338 prélèvements d'eaux souterraines californiennes (Dong et al. 2024),
enrichis par `geoenrich` (section 3) puis ramenés à 97 variables explicatives après
nettoyage — 94 numériques et 3 catégorielles [synthese_resultats.json]. Ces variables
décrivent le contexte du forage (proximité et densité de sites de contamination, sol,
hydrologie, qualité de l'air, co-contaminants), à l'exclusion de toute information de
concentration en PFAS.

#### 4.1.2 Définition de la cible EPA 2024 NPDWR

La cible est binaire : elle vaut 1 lorsqu'au moins une concentration dépasse sa limite
réglementaire individuelle (MCL), ou lorsque l'indice de risque du mélange (HI) dépasse 1
[notebook 10]. Elle est recalculée à partir des concentrations, les valeurs manquantes étant
traitées comme des non-détections (donc sans dépassement). Le jeu obtenu est presque
équilibré : 21 154 prélèvements positifs, soit 45,65 % de l'échantillon
[synthese_resultats.json], ce qui évite d'avoir à recourir à un rééchantillonnage artificiel.

> **Figure —** `fig01_class_distribution.png` : distribution des classes de la cible
> (équilibre positifs / négatifs).

### 4.2 Prétraitement et ingénierie des features

Le prétraitement est appliqué de manière identique aux cinq modèles, à partir du seul jeu
d'entraînement pour éviter toute fuite. Il combine quatre décisions [notebook 10].

D'abord, l'**élimination des fuites** : la cible étant une fonction déterministe des
concentrations, toute variable qui réexprime cette information est retirée — les
concentrations elles-mêmes (`*_ngL`), les indicateurs de détection (`*_detected`,
`label_*`), ainsi que des colonnes proxy comme `gm_dataset_name` (nom du programme de
surveillance, corrélé au protocole d'échantillonnage).

Ensuite, le **seuil de valeurs manquantes** : toute colonne renseignée à moins de 60 % est
écartée, l'imputation au-delà inventant la majorité de la colonne.

> **Figure —** `fig02_missingness.png` : profil des valeurs manquantes des variables
> retenues.

Troisième décision, la plus structurante de ce notebook : la **localisation est retirée des
variables**. Latitude, longitude, comté et découpages administratifs ne sont plus des
features d'aucun modèle. La contamination PFAS étant fortement auto-corrélée dans l'espace,
les conserver laisserait les modèles mémoriser la géographie au lieu d'apprendre des
mécanismes ; les coordonnées ne sont conservées que pour bâtir le graphe et, le cas échéant,
les blocs de la validation croisée spatiale [notebook 10].

Enfin, le **codage** : imputation médiane des numériques, catégorie « Unknown » et encodage
ordinal des catégorielles, avec une standardisation supplémentaire pour le réseau de
neurones, qui exige des entrées centrées-réduites.

Le jeu est ensuite séparé en train, validation et test (60 / 20 / 20) de façon stratifiée,
soit 27 802, 9 268 et 9 268 prélèvements, ce découpage étant partagé par les cinq modèles
[synthese_resultats.json].

### 4.3 Modèles tabulaires de référence

Deux modèles d'arbres servent de référence, car ils représentent l'état de l'art pratique
sur données tabulaires et fixent le niveau de performance à atteindre. La **forêt aléatoire**
(bagging) et **XGBoost** (boosting de gradient) traitent chaque prélèvement comme un vecteur
de variables indépendant des autres. Leurs hyperparamètres sont optimisés par Optuna, et leur
apprentissage est suivi par une courbe d'apprentissage contrôlant le surapprentissage
[notebook 10].

> **Figures —** Random Forest : `fig03_optuna_rf.png` (optimisation Optuna),
> `fig04_rf_learning_curve.png` (courbe d'apprentissage). XGBoost : `fig08_optuna_xgb.png`,
> `fig09_xgb_learning_curve.png`.

### 4.4 Modèle relationnel : HGT (Heterogeneous Graph Transformer)

Le cœur du second volet est un transformeur de graphe hétérogène, qui exploite la
structure de voisinage que les modèles tabulaires ignorent.

#### 4.4.1 Construction du graphe relationnel

Le graphe repose sur de véritables entités partagées plutôt que sur des attributs recopiés
(l'architecture est détaillée au schéma 4). Les nœuds `sample` représentent les prélèvements,
sans aucune variable de localisation. Trois types de nœuds-relais portent le contexte
spatial : les sites de contamination GeoTracker (`source`), les sous-bassins hydrogéologiques
SGMA (`subbasin`) et des clusters géographiques obtenus par KMeans (`geo_cluster`). Des arêtes
`near_geo` relient directement les prélèvements à leurs plus proches voisins spatiaux (KNN).
Deux forages proches d'un même site ou d'un même sous-bassin échangent ainsi de
l'information. Pour éviter toute fuite, clusters, voisinages et sous-bassins sont construits
à l'intérieur de chaque split [notebook 10].

![Schéma 4 — Graphe relationnel HGT](render/latex/schema4.png)

***Schéma 4.*** *Graphe hétérogène relationnel : les nœuds `sample` ne portent aucune
variable de localisation ; ils sont reliés à des entités partagées (`source`, `subbasin`,
`geo_cluster`) et entre eux par des arêtes spatiales `near_geo`. (d'après `notebook 10`)*

#### 4.4.2 Architecture et mécanisme d'attention

Le HGT propage l'information le long des arêtes par un mécanisme d'attention propre à chaque
type de relation : un nœud `sample` agrège différemment les messages venus d'un site, d'un
sous-bassin ou d'un voisin spatial (le calcul d'une couche est illustré au schéma 5). Après
plusieurs couches, chaque prélèvement dispose d'un vecteur de représentation (embedding) qui
résume son contexte relationnel, lequel alimente un classifieur final.

![Schéma 5 — Une couche HGT](render/latex/schema5.png)

***Schéma 5.*** *Mécanisme d'une couche HGT : passage de message et attention propre à chaque
type d'arête, sur le voisinage d'un forage.*

#### 4.4.3 Stratégie d'entraînement

L'entraînement s'étend sur 300 époques avec optimisation Optuna [synthese_resultats.json].
Deux techniques d'augmentation régularisent le modèle : l'ajout de bruit gaussien sur les
variables des nœuds et le retrait aléatoire d'arêtes (DropEdge). Le modèle retenu est celui
qui maximise le F1 sur la validation.

> **Figures —** `fig13_optuna_hgt.png` (optimisation Optuna), `fig14_hgt_learning_curve.png`
> (courbe d'apprentissage).

### 4.5 Modèles hybrides

Deux modèles cherchent à combiner le meilleur des deux familles. L'**Embedding Fusion**
extrait les embeddings appris par le HGT, les réduit par analyse en composantes principales,
puis les concatène aux variables tabulaires avant de les fournir à un XGBoost : il teste si la
représentation relationnelle apporte un complément aux variables de contexte. Le **Stacking**
entraîne un méta-classifieur sur les probabilités produites par la forêt aléatoire, XGBoost et
le HGT, afin d'exploiter leur éventuelle complémentarité [notebook 10].

> **Figures —** Fusion : `fig18_optuna_fusion.png`. Stacking : `fig21_optuna_stacking.png`.

### 4.6 Protocole expérimental

L'évaluation repose sur des métriques adaptées à une cible presque équilibrée mais à enjeu
asymétrique : aire sous la courbe ROC et précision moyenne pour le pouvoir de discrimination,
rappel et précision pour le compromis de décision, F1 et exactitude équilibrée pour une
synthèse. Chaque modèle est optimisé par Optuna sur la validation, et sa robustesse est
contrôlée par une matrice de confusion. Le dispositif principal contre la mémorisation
géographique est le retrait de la localisation des features décrit en 4.2. Une validation
croisée spatiale par blocs (k = 8) est par ailleurs prévue dans le code comme diagnostic de
généralisation hors-zone ; elle est désactivable et n'a pas été exécutée dans le présent
run, ce point étant repris en discussion (section 5). Tous les modèles partagent le même
découpage et le même prétraitement, condition d'une comparaison juste [notebook 10].

---

## 5. Résultats, interprétabilité et discussion

Cette section rapporte les performances des cinq modèles sur le jeu de test, rend leurs
décisions lisibles aux niveaux global et local, ouvre la boîte noire du graphe relationnel,
restitue les prédictions sous forme cartographique, puis discute la portée des résultats. Les
chiffres proviennent du run du 2026-06-27 (mode FULL) [synthese_resultats.json ;
comparaison_modeles.csv].

### 5.1 Performances comparées des cinq modèles

Le tableau ci-dessous résume les performances sur le test (9 268 prélèvements). Les modèles
tabulaires dominent nettement : la forêt aléatoire et XGBoost atteignent une aire sous la
courbe ROC proche de 0,98, et le Stacking les égale sans les dépasser. Le HGT relationnel,
qui ne reçoit la géographie que par la structure du graphe, reste en retrait
(ROC-AUC 0,906), et la Fusion se situe entre les deux familles [comparaison_modeles.csv].

| Modèle | ROC-AUC | AP | Précision | Rappel | F1 | Bal. Acc. |
|---|---|---|---|---|---|---|
| Random Forest | **0,979** | 0,976 | 0,930 | 0,920 | **0,925** | 0,931 |
| Stacking | 0,977 | 0,974 | 0,923 | 0,926 | 0,925 | 0,931 |
| XGBoost | 0,977 | 0,974 | 0,922 | 0,923 | 0,923 | 0,929 |
| Embedding Fusion | 0,960 | 0,954 | 0,886 | 0,887 | 0,887 | 0,896 |
| HGT relationnel | 0,906 | 0,886 | 0,823 | 0,828 | 0,825 | 0,839 |

> **Figures —** `fig24_roc_pr_all.png` (courbes ROC et précision-rappel des cinq modèles),
> `fig25_calibration_all.png` (courbes de calibration), `fig26_confusion_all.png` (matrices de
> confusion), `fig27_dashboard.png` (tableau de bord comparatif).

Trois constats se dégagent. D'abord, retirer la localisation des variables n'effondre pas les
modèles tabulaires : ils restent très performants en s'appuyant sur les variables de contexte
issues de `geoenrich`. Ensuite, structurer la géographie dans un graphe (HGT relationnel) ne
suffit pas à égaler ces variables de contexte : le modèle relationnel capte un signal réel
mais plus pauvre. Enfin, les hybrides ne tirent pas profit du relationnel : la Fusion dilue
légèrement la performance tabulaire, et le Stacking se contente d'égaler la forêt aléatoire,
signe d'une faible complémentarité entre les familles.

### 5.2 Interprétabilité globale : importance et concordance

L'analyse des importances de variables, menée par valeurs de Shapley, identifie les facteurs
qui pèsent le plus dans les prédictions et mesure l'accord entre modèles. Les variables les
plus influentes sont la catégorie du forage, la profondeur de la nappe (`depth_to_water_m`),
la densité de sites GeoTracker dans le voisinage (`n_geotracker_within_10km` et `_50km`) et la
direction d'écoulement [concordance_importance.csv]. Ce sont des grandeurs de contexte
hydrogéologique et de proximité aux sources, et non des coordonnées, ce qui est cohérent avec
le retrait de la localisation. La concordance des rangs entre forêt aléatoire et XGBoost est
forte ; celle du HGT s'en écarte davantage, reflétant qu'il lit l'information par un canal
différent.

> **Figures —** `fig40_importance_par_modele.png` (importances par modèle, Top 25),
> `fig41_concordance_rangs.png` (concordance des rangs). SHAP par modèle :
> `fig06_shap_bar_rf.png`, `fig07_shap_beeswarm_rf.png`, `fig11_shap_bar_xgb.png`,
> `fig12_shap_beeswarm_xgb.png`, `fig16_hgt_importance.png`, `fig20_shap_fusion.png`,
> `fig23_stacking_importance.png`.

### 5.3 Interprétabilité locale

Au-delà des tendances globales, chaque modèle est expliqué au niveau d'une prédiction
individuelle, sur un forage positif et un forage négatif, par décomposition de la décision en
contributions de variables. Ces explications locales rendent une prédiction défendable
auprès d'un décideur : elles montrent quelles caractéristiques du forage ont poussé la
décision vers le dépassement ou vers la conformité.

> **Figures —** Random Forest : `fig06b_shap_local_rf_positif.png`,
> `fig06b_shap_local_rf_negatif.png`. XGBoost : `fig11b_shap_local_xgb_positif.png`,
> `fig11b_shap_local_xgb_negatif.png`. Fusion : `fig20b_shap_local_fusion_positif.png`,
> `fig20b_shap_local_fusion_negatif.png`. Stacking :
> `fig23b_shap_local_stacking_positif.png`, `fig23b_shap_local_stacking_negatif.png`.

### 5.4 Sous le capot du HGT

À des fins pédagogiques, le fonctionnement du graphe relationnel est exposé pas à pas : le
voisinage d'un site de contamination et celui d'un forage donnent à voir la structure
hétérogène effectivement construite, tandis que la projection en deux dimensions des
embeddings appris montre dans quelle mesure le modèle sépare les forages à risque des autres.
Cette lecture explique aussi le retrait du HGT en performance : la séparation apprise est
réelle mais moins tranchée que celle obtenue par les variables de contexte.

> **Figures —** `fig30_site_neighborhood.png` (voisinage d'un site GeoTracker),
> `fig31_well_neighborhood.png` (voisinage d'un forage), `fig32_hgt_embedding.png` et
> `fig33_hgt_embeddings_2d.png` (projection 2D des embeddings).

### 5.5 Cartographie des prédictions

Les prédictions du meilleur modèle sont restituées spatialement, ce qui permet de confronter
les zones jugées à risque à la géographie connue de la contamination et d'identifier
visuellement d'éventuelles concentrations de faux positifs ou négatifs.

> **Figure —** `fig28_map_predictions.png` (carte des prédictions du meilleur modèle).

### 5.6 Discussion

Les résultats appellent une lecture nuancée. Le principal enseignement est que, sur ce jeu de
données, les variables de contexte produites par `geoenrich` portent l'essentiel du signal
prédictif : des modèles tabulaires classiques, privés de toute coordonnée, atteignent une
discrimination élevée. Le pari de ce notebook — porter la géographie par la structure d'un
graphe plutôt que par des variables — n'améliore pas la prédiction : le HGT relationnel reste
en deçà des modèles tabulaires, et les hybrides ne dégagent pas de complémentarité exploitable.
Ce résultat, négatif au sens où l'architecture la plus élaborée n'est pas la plus performante,
a sa valeur : il montre qu'un enrichissement de contexte soigné peut rendre superflue une
modélisation relationnelle plus lourde.

Ces conclusions doivent toutefois être tempérées par une limite d'évaluation importante. Les
performances rapportées reposent sur un découpage stratifié, qui ne teste pas la
généralisation à des régions entièrement non vues. Or la contamination PFAS est fortement
auto-corrélée dans l'espace, et même des variables de contexte (densité de sites, proximité)
encodent indirectement la position : une partie de la performance pourrait donc tenir à une
auto-corrélation spatiale résiduelle plutôt qu'à des mécanismes transférables. La validation
croisée spatiale par blocs, prévue dans le code mais non exécutée dans ce run, est l'outil
qui permettrait de mesurer ce risque ; son exécution constitue la priorité immédiate pour
consolider les conclusions. En l'absence de ce test, les comparaisons entre modèles restent
valides — elles s'effectuent à protocole identique — mais le niveau absolu de performance doit
être interprété comme une borne supérieure optimiste.

---

## 6. Synthèse

Ce chapitre a présenté une double contribution au problème de la prédiction du risque PFAS
dans les eaux souterraines californiennes. La première est un outil, `geoenrich`, qui
transforme une logique d'enrichissement monolithique et liée au cas PFAS en un moteur
générique, organisé en trois couches et fondé sur un contrat d'entrée minimal séparant le
*où/quand* universel de la charge utile étudiée. Sa correction a été établie par un test de
parité avec le pipeline d'origine, et son indépendance vis-à-vis du programme GAMA par
inspection du code et par conception. La seconde contribution est un banc de cinq modèles —
deux modèles tabulaires de référence, un transformeur de graphe hétérogène et deux modèles
hybrides — évalués à protocole identique sur un dataset enrichi de plus de quarante mille
prélèvements, avec pour cible le dépassement du seuil réglementaire EPA 2024.

L'enseignement principal des expériences est que les variables de contexte produites par
`geoenrich` portent l'essentiel du signal prédictif : des modèles tabulaires privés de toute
coordonnée atteignent une discrimination élevée, tandis que le pari de structurer la
géographie dans un graphe relationnel n'améliore pas la prédiction et que les hybrides ne
dégagent pas de complémentarité. Ce résultat, où l'architecture la plus élaborée n'est pas la
plus performante, souligne la valeur d'un enrichissement de contexte soigné, qui est
précisément l'objet du premier volet.

Deux limites encadrent ces conclusions et tracent les perspectives. D'une part,
l'indépendance de `geoenrich` reste établie par construction et demande à être démontrée
empiriquement en faisant passer par l'outil un dataset d'un autre polluant ou d'une autre
région. D'autre part, l'évaluation des modèles repose sur un découpage qui ne teste pas la
généralisation à des zones entièrement non vues ; l'exécution de la validation croisée
spatiale par blocs, déjà prévue dans le code, est la priorité immédiate pour distinguer la
part de signal transférable de l'auto-corrélation spatiale résiduelle. Ces deux chantiers
prolongent naturellement le travail présenté, sans en remettre en cause les acquis : un outil
d'enrichissement réutilisable et une comparaison rigoureuse et interprétable de modèles de
prédiction de la contamination.

---

## Sources

**Documents internes au projet (espace de travail)**

- `geoenrich/RAPPORT_OUTIL.md` — rapport d'outil : livrables, architecture en trois
  couches, indépendance vis-à-vis de GAMA, taille du module `merge.py` d'origine.
- `geoenrich/README.md` — contrat d'entrée, bibliothèque d'opérateurs, principe
  charge utile / contexte.
- `geoenrich/PARITY.md` — validation de parité (77/94 colonnes identiques, 17 mieux
  remplies, 0 régression).
- `geoenrich/configs/california.yaml` — instanciation californienne : couches de
  référence et opérateurs mobilisés.
- `ca-pfas-ml/notebooks/10_pfas_5_modeles_HGT_relationnel.ipynb` — **notebook livrable
  retenu** : cible EPA 2024 NPDWR, prétraitement (anti-fuite, localisation retirée des
  features), banc des cinq modèles, graphe relationnel, protocole d'évaluation.
- `ca-pfas-ml/livrables/10/PFAS_HGT_relationnel_outputs_20260627_1230/` — **livrable
  d'exécution** (run du 2026-06-27, mode FULL) :
  - `results/synthese_resultats.json` — dimensions du jeu (46 338 lignes, 97 variables,
    45,65 % positifs), découpage 60/20/20, métriques des cinq modèles.
  - `results/comparaison_modeles.csv`, `results/concordance_importance.csv` — tableaux
    comparatifs et concordance des importances.
  - `figures/figXX_*.png` — figures référencées dans le texte par leur nom de fichier.

**Références externes**

- **Dong et al. 2024** — jeu de données CA-PFAS-ASGWS : mesures de PFAS dans les eaux
  souterraines de Californie (support empirique du travail). *[Référence
  bibliographique complète à compléter.]*
- **EPA 2024 NPDWR** — U.S. Environmental Protection Agency, *National Primary Drinking
  Water Regulation* pour les PFAS (2024) : limites maximales par molécule et indice de
  risque pour les mélanges. *[Référence bibliographique complète à compléter.]*

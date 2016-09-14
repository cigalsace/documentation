**Coopération pour l'Information Géographique en Alsace**

# Recherche d’optimisation pour la publication web de données Raster sur Geoserver via GDAL dans un contexte geOrchestra

<!-- TOC depthFrom:2 depthTo:3 withLinks:1 updateOnSave:0 orderedList:0 -->

- [Avant-Propos](#Avant-Propos-)
- [Définitions](#définitions-)
	- [Base de données](#base-de-données-)
	- [Fiche de métadonnées](#fiche-de-métadonnées-)
- [Principes de base](#principes-de-base-)
- [Procédure](#procédure-)
	- [Placer les fichiers ressources sur le serveur CIGAL](#placer-les-fichiers-ressources-sur-le-serveur-cigal-)
	- [Décrire le jeu de données](#décrire-le-jeu-de-données-)
	- [Exporter la fiche descriptive au format XML](#exporter-la-fiche-descriptive-au-format-xml-)
	- [Déposer la fiche au format XML sur le serveur CIGAL](#déposer-la-fiche-au-format-xml-sur-le-serveur-cigal-)

<!-- /TOC -->

## Avant-Propos <a id="Avant-Propos-"></a>

La préparation de données Raster est fonction de compromis entre espace disque disponible, maturité des outils à disposition et public ciblé (performance et qualité de rendu).

Une donnée livrée est rarement directement prête pour être servie de manière optimisée en flux pour trois raisons :

-	Les volumétries des données brutes peuvent se révélées très importantes et les espaces disques serveurs onéreux
-	Les zones de bordure peuvent nécessiter une préparation particulière pour un affichage propre (canal alpha de transparence, footprint, nodata ou sld)
-	Modifier des paramètres avancés tels que le tuilage interne (inner tiling), les aperçus (overview) ou la taille des dalles (merge) peut influencer les temps de réponse.

Avec Geoserver l’administrateur de donnée dispose de toute une gamme de formats en entrée.

Cette note a pour vocation de capitalisé les éléments qui justifie les choix en matière de préparation de Raster sur la géoplateforme CIGAL.


## Retour sur la compression raster <a id="#Compression-"></a>

Performances et volumétrie pourront fluctuer de manière importante selon le mode de compression retenu.

|      Compression         |      Commentaire     |
|----------|--------------|
|ecw|Avec ou sans perte. Format propriétaire qui nécessite une licence pour la publication web. Il ne sera pas traité dans la suite du document.|
|Jp2000|Avec ou sans perte|
|tif LZW|Sans perte|
|tif deflate|Sans perte|
|tif jpeg|Avec perte|

Cf littérature web pour de plus amples information sur les algorithmes utilisés

A titre d’exemple ci-dessous les variations en volumétrie d’une ortho RVB à 20cm de résolution :

|      Format         |      Taille proportion     |      Volume sur 1 Département     |
|----------|--------------|--------------|
|tif (livraison brute)|100%|412 Go|
|tif tiling overview|142%|585 Go|
|tif lzw tiling overview|137%|565 Go|
|tif deflate tiling overview|110%|453 Go|
|tif deflate alpha tiling overview|119%|490 Go|
|tif jpeg tiling overview|27%|111 Go|
|tif jpeg alpha tiling overview|40%|165 Go|

(Ces résultats ont été obtenus dans les taux de compression par défaut et en rajoutant les inner tiling et overview pour préparer à la publication)

````
gdal_translate -a_srs EPSG:3948 -co COMPRESS=DEFLATE -co TILED=YES -co BLOCKXSIZE=512 -co BLOCKYSIZE=512 input.tif output.tif
gdaladdo --config COMPRESS_OVERVIEW DEFLATE --config GDAL_TIFF_OVR_BLOCKSIZE 512 output.tif 2 4 8 16 32 64 128

````

Si l’on souhaite rajouter la bande alpha au fichier il faut prévoir environ 10% de place en plus

A noter que nous ne considérons pas ici les possibilités de varier l’option PREDICTOR en horizontal ou floating qui donne de meilleurs résultats mais qui semble mal géré par Geoserver.

De la même manière nous avons constaté dans nos tests que les overview externes sont mal supportés par Geoserver

Pour les scripts batch de préparation se référer à
https://github.com/cigalsace/processes/tree/master/gdal


### 2.	Confrontation <a id="#Confrontation-"></a>

D’après

http://geoserver.geo-solutions.it/edu/en/enterprise/index.html

https://wiki.osgeo.org/wiki/Banc_d'essai_comparatif

http://www.digital-geography.com/geotiff-compression-comparison/#.V616dTUl8cN

Des tests basiques jmeter nous ont permis d'aboutir aux résultats suivants
https://github.com/cigalsace/processes/tree/master/jmeter

Dans l'ordre des cartouches:

| Data | Size Go | prepa |
|----------|------------|-----------|
|ecw |0.621 |none 3948|
|tifa deflate|8.4 |mosa overview inner tiling 3948|
|tifa lzw|11|mosa overview inner tiling 3948|
|tif deflate|7.7|mosa overview inner tiling 3948|
|tifa jpeg|2.8|mosa overview inner tiling 3948|
|tifa deflate|7.5|pyramid inner tiling 3948|

https://www.cigalsace.org/geoserver/test_public/ows?SERVICE=WMS&VERSION=1.3.0&FORMAT=image/jpeg&REQUEST=GetMap&SRS=EPSG:3948
![graphique agrege](https://cloud.githubusercontent.com/assets/5012040/17619503/077ddd88-6086-11e6-97b3-2818cd9569cb.png)
![graphique evolution temps de reponses](https://cloud.githubusercontent.com/assets/5012040/17619505/098a7f6e-6086-11e6-990d-d05a0e2176f7.png)

https://www.cigalsace.org/geoserver/test_public/ows?SERVICE=WMS&VERSION=1.3.0&FORMAT=image/png&REQUEST=GetMap&SRS=EPSG:3948
![graphique agrege](https://cloud.githubusercontent.com/assets/5012040/17619625/e7af51f2-6086-11e6-8103-fa08c6535ff3.png)
![graphique evolution temps de reponses](https://cloud.githubusercontent.com/assets/5012040/17619629/ebb7d490-6086-11e6-8f0d-50d98ca15a6a.png)

https://www.cigalsace.org/geoserver/test_public/ows?SERVICE=WMS&VERSION=1.3.0&FORMAT=image/jpeg&REQUEST=GetMap&SRS=EPSG:3857
![graphique agrege](https://cloud.githubusercontent.com/assets/5012040/17619640/fe60e1fe-6086-11e6-8a21-1d55841417ea.png)
![graphique evolution temps de reponses](https://cloud.githubusercontent.com/assets/5012040/17619641/ffcc706c-6086-11e6-9b38-5de6a2dd2185.png)

https://www.cigalsace.org/geoserver/test_public/ows?SERVICE=WMS&VERSION=1.3.0&FORMAT=image/png&REQUEST=GetMap&SRS=EPSG:3948
![graphique agrege](https://cloud.githubusercontent.com/assets/5012040/17619644/067be5aa-6087-11e6-86d0-a0ccb603d50e.png)
![graphique evolution temps de reponses](https://cloud.githubusercontent.com/assets/5012040/17619646/07e746a0-6087-11e6-9ddd-501b6081d562.png)

La version 15.12 de georchestra affiche des anomalies avec le format ecw

On voit que le tif deflate donne de bons résultats pour un compromis entre volumétrie et performance.

Les entrepots de type image mosaique affichent une meilleure réaction

Les BigTiff répondent vraiment bien (ici avec granule d'environ 10 Giga + overview externe)

![graphique agrege](https://cloud.githubusercontent.com/assets/5012040/17694608/f14ee242-63a4-11e6-908c-1a781890d712.png)
![graphique evolution temps de reponses](https://cloud.githubusercontent.com/assets/5012040/17694611/f2f2d64e-63a4-11e6-9aab-976b0485482c.png)



## Analyse conclusions pour la géoplateforme CIGAL <a id="#Analyse"></a>



Le tif non compressé est le format qui apporte les meilleures performances. D’après le tableau du 1.2 une ortho 20cm en tif deflate sur 10 départements prend 4.5 To, un équilibre est donc à trouver avec les différentes méthodes et niveaux de compressions.

Dans le cas de la géoplateforme CIGAL nous retenons pour le moment les pistes suivantes:
-Privilégier deflate à lzw
-Image mosaique avec overview interne plutôt que image pyramide (pour des produits types orthophoto HR départementale)
-En général nous réservons la qualité optimale (sans perte) à la dernière ortho (par exemple la 2015) et les milésimes plus anciens, les produits dérivés (infrarouge…) ou les ortho agglo haute résolution (8cm) sont compresser en JPEG 2000 (voir 4.1 pour des exemples de dégradation)
-Le développement de l’Open Data devrait rendre possible de plus en plus un accès direct à la donnée brute en téléchargement. Cela pourra décharger dans certain cas l’utilisation des flux de téléchargement type WCS qui par ailleurs impactent fortement les serveurs.

D’après
http://fr.slideshare.net/geosolutions/geoserver-on-steroids-foss4g-2015

Reste également à jouer sur les par
(slide 11) Des optimisations sont à trouver dans les paramètres Geoserver selon les caractéristiques serveur

Les reprojections Geoserver ne semble pas jouer de manière significative sur la performance ce qui nous incite à privilégier l93 pour les formats de publication.

Utilisation Geowebcache pour WMTS dans les gridset les plus courament utilisés

Privilégier dans les viewer les appels image/jpeg (10x moins lourds), attention également à la taille des tuiles


## Procédure <a id="procédure-"></a>

### Placer les fichiers ressources sur le serveur CIGAL <a id="#placer-les-fichiers-ressources-sur-le-serveur-cigal-"></a>

Pour vous connecter à Pydio, rendez-vous à l'adresse : <https://www.cigalsace.org/files> Si vous n'êtes pas encore authentifié, saisissez votre identifiant et votre mot de passe avant de valider.

![login](img/login.jpg)

Une fois dans Pydio, sélectionnez dans la liste de gauche votre dépôt de métadonnées. Son nom est de la forme « Metadata_ORG » où « ORG » correspond au nom ou au sigle de votre organisme.

![pydio1](img/pydio1.jpg)

Il vous est ensuite possible :
- De créer des répertoires pour organiser vos fichiers (bouton « créer » en haut à droite)
- De déposer des fichiers par simple glisser/déposer sur l'écran (bouton « Transférer » en haut à droite)

![pydio2](img/pydio2.jpg)

Les fichiers ainsi déposés sur la plateforme CIGAL sont alors disponibles via une adresse du type « <https://www.cigalsace.org/metadata/ORG/REP/ressource.ext> », où :
- « ORG » est le nom de la structure ou son sigle (identique au « ORG » du nom de dépôt Pydio)
- « REP » est, le cas échéant, le chemin vers les fichiers dans le dépôt sur Pydio
- « ressource.ext » est le nom du fichier de ressource concerné.

**_Recommandations :_**
- _Créer un répertoire pour chaque type de ressources transversales comme par exemple les logos qui sont utilisés par différentes fiches descriptives._
- _Créer un répertoire par jeu de données pour y placer les éléments spécifiques (illustration, document technique, liste des attributs de la couche, etc.)_

### Décrire le jeu de données <a id="#décrire-le-jeu-de-données-"></a>

La description du jeu de données peut être réalisée assez simplement directement en ligne via mdEdit. L'accès à cette application ne nécessite pas d'authentification. Il vous suffit de vous rendre à l'adresse suivante : <https://www.cigalsace.org/tools/mdEdit/>

_A la première connexion, en cas de problème d'affichage, il peut être nécessaire de rafraîchir la page !_

![mdedit1](img/mdedit1.jpg)

Il vous suffit alors de compléter le formulaire à partir des informations dont vous disposez sur le jeu de données. Pour plus de détail sur les informations attendues, vous pouvez vous rapportez au « Guide simplifié de saisie des métadonnées CIGAL ».

**_Liens vers les ressources :_**
_Les liens vers les ressources déposées sur la plateforme via Pydio doivent être renseignés selon une URL de la forme « <https://www.cigalsace.org/metadata/ORG/REP/ressource.ext> » comme décrit précédemment._

### Exporter la fiche descriptive au format XML <a id="#exporter-la-fiche-descriptive-au-format-xml-"></a>

Pour exporter la fiche descriptive au format XML, utiliser le bouton en haut à droite.

![mdedit2](img/mdedit2.jpg)

mdEdit permet également de recharger une fiche descriptive au format XML :
- Pour la consulter
- Pour la compléter ou la mettre à jour

Il est ainsi possible de créer une fiche partiellement remplie et de la réutiliser comme un modèle pour saisir de nouvelles fiches.

### Déposer la fiche au format XML sur le serveur CIGAL <a id="#déposer-la-fiche-au-format-xml-sur-le-serveur-cigal-"></a>

Le dépôt du fichier XML sur le serveur CIGAL se fait via Pydio selon le même principe que le dépôt des fichiers ressources expliqué ci-dessus.

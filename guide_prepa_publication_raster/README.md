**Coopération pour l'Information Géographique en Alsace**

# Recherche d’optimisation pour la publication web de données Raster sur Geoserver via GDAL dans un contexte geOrchestra

<!-- TOC depthFrom:2 depthTo:3 withLinks:1 updateOnSave:0 orderedList:0 -->

- [Avant-Propos](#avant-propos-)
- [Compression Raster](#compression-raster-)
- [Confrontation](#confrontation-)
- [Conclusions](#conclusions-)

<!-- /TOC -->

## Avant-Propos <a id="avant-propos-"></a>

La préparation de données Raster est fonction de compromis entre espace disque disponible, maturité des outils à disposition et public ciblé (performance et qualité de rendu).

Une donnée livrée est rarement directement prête pour être servie de manière optimisée en flux pour trois raisons :

-	Les volumétries des données brutes peuvent se révélées très importantes et les espaces disques serveurs onéreux
-	Les zones de bordure peuvent nécessiter une préparation particulière pour un affichage propre (canal alpha de transparence, footprint, nodata ou sld)
-	Modifier des paramètres avancés tels que le tuilage interne (inner tiling), les aperçus (overview) ou la taille des dalles (merge) peut influencer les temps de réponse.

Avec Geoserver l’administrateur de donnée dispose de toute une gamme de formats en entrée.

Cette note a pour vocation de capitaliser les éléments justifiant les choix en matière de préparation de Raster sur la géoplateforme CIGAL.


## Compression raster <a id="compression-raster-"></a>

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

| Format                        | Taille proportion | Volume sur 1 département |
|-------------------------------|-------------------|--------------------------|
| tif(livraisonbrute)           | 100%              | 412Go                    |
| tiftilingoverview             | 142%              | 585Go                    |
| tiflzwtilingoverview          | 137%              | 565Go                    |
| tifdeflatetilingoverview      | 110%              | 453Go                    |
| tifdeflatealphatilingoverview | 119%              | 490Go                    |
| tifjpegtilingoverview         | 27%               | 111Go                    |
| tifjpegalphatilingoverview    | 40%               | 165Go                    |

(Ces résultats ont été obtenus dans les taux de compression par défaut et en rajoutant les inner tiling et overview pour préparer à la publication)

````
gdal_translate -a_srs EPSG:3948 -co COMPRESS=DEFLATE -co TILED=YES -co BLOCKXSIZE=512 -co BLOCKYSIZE=512 input.tif output.tif
gdaladdo --config COMPRESS_OVERVIEW DEFLATE --config GDAL_TIFF_OVR_BLOCKSIZE 512 output.tif 2 4 8 16 32 64 128

````

Si l’on souhaite rajouter la bande alpha au fichier il faut prévoir environ 10% de place de stockage en plus

A noter que nous ne considérons pas ici les possibilités de varier l’option PREDICTOR en horizontal ou floating qui donne de meilleurs résultats mais qui semble mal gérée par Geoserver 2.8

De la même manière nous avons constaté dans nos tests que les overview externes ne seraient pas supportées par Geoserver 2.8

Pour les scripts batch de préparation se référer à
https://github.com/cigalsace/processes/tree/master/gdal


### Confrontation <a id="confrontation-"></a>

D’après

http://geoserver.geo-solutions.it/edu/en/enterprise/index.html

https://wiki.osgeo.org/wiki/Banc_d'essai_comparatif

http://www.digital-geography.com/geotiff-compression-comparison/#.V616dTUl8cN

Ainsi ques des tests basiques jmeter
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

Geoserver 2.8 supporte mal le format ecw

On voit que le tif deflate donne de bons résultats pour un compromis entre volumétrie et performance.

Ci-dessous, les BigTiff répondent vraiment bien (ici avec granule d'environ 10 Giga + overview externe) car moins de fichiers à ouvrir.

![graphique agrege](https://cloud.githubusercontent.com/assets/5012040/17694608/f14ee242-63a4-11e6-908c-1a781890d712.png)
![graphique evolution temps de reponses](https://cloud.githubusercontent.com/assets/5012040/17694611/f2f2d64e-63a4-11e6-9aab-976b0485482c.png)

Les entrepots de type image mosaique réagissent mieux. De plus, ils proposent maintenant une fonctionalité footprints qui permettrait de faire l'économie de la bande alpha. Par contre nous identifions un problème répertorié ici
https://osgeo-org.atlassian.net/browse/GEOS-6760?attachmentViewMode=list
Dans le cas d'overview interne il n'est pas possible pour le moment d'appliquer un footprints couvrant des dalles non pleines. Fonctionnalité décrite ici
http://docs.geoserver.org/2.8.x/en/user/tutorials/imagemosaic_footprint/imagemosaic_footprint.html#footprint-configured-with-footprints-shp

Enfin, appeler un WMS dans une autre projection différente de celle native de publication ne semble pas trop impactante par rapport aux temps de retour.


## Conclusions <a id="conclusions"></a>

Le tif non compressé est le format qui apporte les meilleures performances. Cependant, une ortho 20cm en tif deflate sur 10 départements pèserait 4.5 To. Un équilibre est donc à trouver.

Dans le cas de la géoplateforme CIGAL nous retenons pour le moment les pistes suivantes:

- Privilégier deflate à lzw
- Image mosaique avec overview interne plutôt que image pyramide (pour des produits types orthophoto HR départementale) et travailler à faire fonctionner convenablement le footprints
- En général nous réservons la qualité optimale (sans perte) à la dernière ortho (par exemple la 2015) et les milésimes plus anciens, les produits dérivés (infrarouge…) sont compressées en JPEG 2000
- L'axe par téléchargement FTP des dalles type Open Data serait privélégié au développement du WCS consommateur pour les servers.
- Le l93 serait prioritaire comme format de publication.

Resterait également:

- à jouer sur les pararamètres Geoserver
http://fr.slideshare.net/geosolutions/geoserver-on-steroids-foss4g-2015
(slide 11)
- à exploiter Geowebcache WMTS dans les gridset les plus courament utilisés
- à encourager pour les viewers les appels image/jpeg sur de petites tuiles.

**Coopération pour l'Information Géographique en Alsace**

# Tutoriel pour la publication web de données Raster sur Geoserver via GDAL dans un contexte geOrchestra

<!-- TOC depthFrom:2 depthTo:3 withLinks:1 updateOnSave:0 orderedList:0 -->

- [Image pyramide](#pyra-)
- [Image mosa](#mosa-)
- [Geoserver](#geoserver-)

<!-- /TOC -->

Justification des choix dans https://github.com/cigalsace/documentation/tree/master/guide_prepa_publication_raster

Eléments de comparaison entre les deux méthodes (de 1 à 5)

| Eléments de comparaison sur une ortho départementale | Image mosaique | Image pyramide |
|----------|------------|-----------|
|performance à grande échelle |4 |3|
|Ortho départementale performance à petite échelle|1 |4|
|Qualité du rendu aux échelles intermédiaires|2|3|
|Volume espace disque|3|2|
|Complexité de la préparation|1|3|
|Total|11|15|

**Les dernières publications GéoGrandEst se font en image pyramide, le chapitre sur les images mosaique est laissé pour mémoire**

## Image pyramide <a id="pyra-"></a>

Pour chaque dalle on calcul le contour et on défini la valeur nodata

```
for %%G IN (input\*.tif) DO (
gdaltindex -t_srs EPSG:2154 index\%%~nG.shp %%G
gdalwarp --config GDALWARP_IGNORE_BAD_CUTLINE YES -t_srs EPSG:2154 -dstnodata -q -cutline index\%%~nG.shp -crop_to_cutline -of GTiff %%G output\%%~nG.tif
```

On construit le vrt

```
for %%i in (output\%%~nG.tif) do echo %%i >> index\liste.txt
gdalbuildvrt -input_file_list index\liste.txt index\liste.vrt
```

Puis la pyramide

```
gdal_retile -v -levels 5 -ps 16384 16384 -s_srs EPSG:2154 -r bilinear -co COMPRESS=DEFLATE -co TILED=TRUE -co INTERLEAVE=BAND -co BLOCKXSIZE=256 -co BLOCKYSIZE=256 -targetDir pyr index\liste.vrt
```

## Image mosaique <a id="mosa-"></a>

### Etape 1: Index et nouvelle maille

Tout d'abord, reconstituons l'index des dalles livrées

```gdaltindex 1_1.shp data\*.tif```

L'objectif est ici de redéfinir de nouvelles dalles entre 1 et 10Go en tif non compressé, par exemple ici en 10/10km

Pour cela nous utilisons

*vecteur/outils de recherche/grille vecteur*

Attention à ce que le projet soit bien dans la même projection que le carroyage

![img1](img/10_10.png)

Les identifiants des nouvelles dalles sont crées à l'aide de la calculatrice de champs QGIS

```concat(  left(  "X_MIN" ,4), '_',left(  "Y_MIN" ,4))```

![img2](img/location.png)

### Etape 2: Merge raster <a id="etape2-"></a>

L'étape suivante consiste à calculer les centroides de la maille d'origine 1_1.shp

*vecteur/outils de géométrie/centroïdes de polygones*

Puis de récupérer pour chaque point les identifiants des dalles 10/10

*vecteur/outils de gestion de données/joindre les attributs par localisation*

![img2](img/centro.png)

Copier coller la nouvelle table d'attribut dans un tableur afin de ne garder que la correspondance

|      10_10         |      1_1     |
|----------|--------------|
|1986_7302|\\path\1988_7308_C48.tif[espace]|
|1986_7302|\\path\1988_7308_C48.tif[espace]|
|...|...|

La macro excel transpo.xls https://github.com/cigalsace/documentation/blob/master/tuto_prepa_publication_raster/transpo.xls nous permet de regrouper les dalles par paquet

On retravaille le résultat par concaténation de manière à obtenir pour chaque dalle

```call gdal_merge -init 255 -o output.tif input1_1.tif input1_2.tif... ```

Formule xls
=CONCATENER("call gdal_merge -init 255 -o path:\\";B1;".tif "; C1;D1;E1;F1;G1;H1;I1;J1;K1;L1;M1;N1;O1;P1;Q1;R1;S1;T1;U1;V1;W1;X1;Y1;Z1;AA1;AB1;AC1;AD1;AE1;AF1;AG1;AH1;AI1;AJ1;AK1;AL1;AM1;AN1;AO1;AP1;AQ1;AR1;AS1;AT1;AU1;AV1;AW1;AX1;AY1;AZ1;BA1;BB1;BC1;BD1;BE1;BF1;BG1;BH1;BI1;BJ1;BK1;BL1;BM1;BN1;BO1;BP1;BQ1;BR1;BS1;BT1;BU1;BV1;BW1;BX1;BY1;BZ1;CA1;CB1;CC1;CD1;CE1;CF1;CG1;CH1;CI1;CJ1;CK1;CL1;CM1;CN1;CO1;CP1;CQ1;CR1;CS1;CT1;CU1;CV1;CW1;CX1;CY1;CZ1...)

=CONCATENER("call gdal_merge -init 255 -o E:\raster\ortho_CA\output52\";B1;".tif "; C1;D1;E1;F1;G1;H1;I1;J1;K1;L1;M1;N1;O1;P1;Q1;R1;S1;T1;U1;V1;W1;X1;Y1;Z1;AA1;AB1;AC1)

=CONCATENER(Résultat!A1;Résultat!AD1;Résultat!AE1;Résultat!AF1;Résultat!AG1;Résultat!AH1;Résultat!AI1;Résultat!AJ1;Résultat!AK1;Résultat!AL1;Résultat!AM1;Résultat!AN1;Résultat!AO1;Résultat!AP1;Résultat!AQ1;Résultat!AR1;Résultat!AS1;Résultat!AT1;Résultat!AU1;Résultat!AV1;Résultat!AW1;Résultat!AX1;Résultat!AY1;Résultat!AZ1;Résultat!BA1;Résultat!BB1)

=CONCATENER(A1;Résultat!BC1;Résultat!BD1;Résultat!BE1;Résultat!BF1;Résultat!BG1;Résultat!BH1;Résultat!BI1;Résultat!BJ1;Résultat!BK1;Résultat!BL1;Résultat!BM1;Résultat!BN1;Résultat!BO1;Résultat!BP1;Résultat!BQ1;Résultat!BR1;Résultat!BS1;Résultat!BT1;Résultat!BU1;Résultat!BV1;Résultat!BW1;Résultat!BX1;Résultat!BY1;Résultat!BZ1;Résultat!CA1;Résultat!CB1;Résultat!CC1)

=CONCATENER(B1;Résultat!CD1;Résultat!CE1;Résultat!CF1;Résultat!CG1;Résultat!CH1;Résultat!CI1;Résultat!CJ1;Résultat!CK1;Résultat!CL1;Résultat!CM1;Résultat!CN1;Résultat!CO1;Résultat!CP1;Résultat!CQ1;Résultat!CR1;Résultat!CS1;Résultat!CT1;Résultat!CU1;Résultat!CV1;Résultat!CW1;Résultat!CX1)

### Etape 3: Préparation gdal<a id="etape3-"></a>

Nous utilisons la compression deflate en jouant également sur le inner tiling

```gdal_translate -a_srs EPSG:3948 -co COMPRESS=DEFLATE -co BIGTIFF=YES -co "TILED=YES" -co "BLOCKXSIZE=512" -co "BLOCKYSIZE=512" input.tif output.tif```

(remarque -co bigtiff=yes seulement si vos fichiers dépassent les 4 GB)
(l'option PREDICTOR est mal supporté par GS)

Avant de finir en calculant les overview

```gdaladdo -r average --config COMPRESS_OVERVIEW DEFLATE --config GDAL_TIFF_OVR_BLOCKSIZE 512 E:\raster\ortho11_12\%%~nG_O.tif 2 4 8 16 32 64 128```

(Geoserver supporte mal les aperçus externes
  Pour les aperçus internes de GeoTIFF (ou les aperçus externes au format GeoTIFF), notez que -clean ne réduit pas le fichier. Une exécution ultérieure de gdaladdo avec des niveaux d'aperçu entraînera l'extension du fichier plutôt que la réutilisation de l'espace des vues précédentes supprimées. Si vous souhaitez simplement modifier la méthode de rééchantillonnage sur un fichier qui a déjà des aperçus calculés, vous n'avez pas besoin de nettoyer les aperçus existants.)

Voir https://github.com/cigalsace/processes/blob/master/gdal/RVB_mosa_BIGTIF.bat pour la procédure batchée

## Divers Geoserver<a id="geoserver-"></a>

Quelques liens
http://www.slideshare.net/geosolutions/geoserver-on-steroids-foss4g-2015
http://docs.geoserver.org/stable/en/user/geowebcache/
http://northstar-www.dartmouth.edu/doc/idl/html_6.2/Image_Tiling.html
https://agile-online.org/conference_paper/cds/agile_2012/proceedings/papers/paper_loechel_caching_techniques_for_high-performance_web_map_services_2012.pdf

Quelques considérations

Il semblerait qu’il soit déconseillé d’utiliser le module JP2
http://www.gdal.org/frmt_jp2kak.html

Il est fortement déconseillé de travailler avec ECW sous Geoserver

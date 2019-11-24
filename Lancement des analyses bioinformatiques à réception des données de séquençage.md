# Lancement des analyses bioinformatiques à réception des données de séquençage

## Analyse bioinformatique des données provenant d'Integragen

### Récupérer les données provenant d'Integragen
Les échantillons sont envoyés à des prestataires de séquençage, qui envoient les données de séquençage sous forme de FASTQ via SFTP ou disques durs.

1. Réception d'un mail par Integragen avec informations de connexion et numéro de la PJ.

2. Se connecter au CCUB. Dans `/work/gad/shared/analyse`, créer un nouveau répertoire qui va contenir les données brutes récupérées. Donner au répertoire le nom de la PJ.

3. Se connecter au serveur d'Integragen et récupérer les données. Dans incoming, choisir la bonne PJ.
```bash
sftp duffourd@data.integragen.com
cd incoming
get nom_de_répertoire
exit
```

4. Vérifier les hashs.
```sh
md5sum -c md5sum.txt
```
### Lancer le pipeline
1. Renommer les fichiers. Le numéro GAD figure dans le nom des FASTQ. Notre pipeline ne prend pas en compte ce format de fichier car Integragen change souvent le formatage du nom.

La commande `rename` permet de renommer dans un répertoire toutes les occurrences d'une chaîne de caractère dans les noms de fichiers.
```sh
rename ".fastq" "_fastq" *
```

2. Permettre au pipeline de se connecter à LabKey
Lors du premier lancement du pipeline, vérifier que le fichier `~/.labkeycredentials.txt`. Sinon, le récupérer dans le `/home` de Yannis.

```sh
cp /user1/gad/ya0902du/.labkeycredentials.txt ~/.
```

3. Lancer le pipeline.

```sh
qsub -v ANALYSISDIR=`pwd`,CONFIGFILE=/work/gad/shared/pipeline/2.5.0/common/analysis_config.tsv /work/gad/shared/pipeline/2.5.0/wes_pipeline/full_wes.sh
```

Les FASTQ font être renommés en dijex... (rename_fastq.log).
Plusieurs fichiers de log vont être créés :
- rename_fastq.log
- organize_data_folder.log
- dijex4962/dijex4962.info
- PED7304.family.list permet de savoir si pool, trio, solo...
- run_check_trio.2019-11-21_16-32-06.log pour savoir ce qui a été fait pour les apparentés
- full.wes... liste toutes les commandes envoyées dans qsub
Les données sont disponibles habituellement en quinze heures.

## Archiver les données
Si l'on efface un fichier dans `/work`, celui-ci est effacé et ne peut pas être récupéré s'il n'a pas été préalablement archivé.
Les scripts d'archive ne doivent pas être lancés dans qsub (pour l'instant).

```sh
nohup python /work/gad/shared/pipeline.2.5.0/common /archive_data.py --inputdir `pwd` --qc  /work/gad/shared/vcf/qc --vcf /work/gad/shared/vcf/individu --gvcf /archive/gad/shared/gvcf/ --cnv /work/gad/shared/vcf/individu/ --cnvbatch /archive/gad/shared/cnv --md5 --logfile archive.log --force --nocheck --bammito /archive/gad/shared/bm_GRCh38/ --bam /archive/gad/shared/bam_new_genome_temp/ --fastq /archive/gad/shared/fastq/ &
```

 - `--force` : si un fichier existe dans le répertoire de destination avec même nom, cet argument va l'écraser.
 - `--nocheck` : permet de ne pas vérifier ce qui se trouve dans le répertoire.

## Mettre à jour LabKey
Table suivi exome, `AssayID`
1. Vérifier que l'on a bien analysé tous les dijex reçus.
2. Pour mettre à jour LabKey par batch : sélectionner tous, export en Excel 2007. Cette étape est optionnelle : les modifications suivantes peuvent être réalisées directement dans LabKey.
3. Dans le tableau, changer le statut `envoyé` en `analysé`
4. Dans le tableau, déclarer la date de réception et d'analyse
  - format de cellule : texte
  - formatage de date : aaaa-mm-jj
5. Dans le tableau, attribuer les lecteurs
Lecteur 1 : ALB et AV ; 50/50
Lecteur 2 : CP (100 %), FTMT (100 %), ASDP (80 %), AS (80 %), ALMB 1 par batch, PC (cas particuliers), CT (cas particuliers)
Attention, subtilité des trios. Bien attribuer une famille entière à un lecteur.
6. Dans le cas où l'on a réalisé les modifications dans un tableau exporté, il faut l'importer dans LabKey :
`Reimport run` ; `next` ; nom d'AssayID : le même ; `upload datafile` et choix du fichier ; `save and finish`.
7. Retourner dans la table suivi exome ; filtre sur `Assay_ID` ; cocher toutes les lignes et cliquer sur `recall` (pour supprimer les lignes).
8. Revenir à la feuille d'importation, cocher toutes les lignes et cliquer sur `copy to study` (ce qui va copier les données dans la table finale).
9. Envoyer un mail aux lecteurs pour les prévenir que de nouvelles données disponibles à la lecture.

## Récupérer les données du CNRGH
1. Réception d'un mail de Delphine Bacq.

2. Créer un nouveau répertoire pour l'analyse
```sh
mkdir /work/gad/shared/analyse/nom_de_répertoire
cd /work/gad/shared/analyse/nom_de_répertoire
cp ~ya0902du/bin/get_data_rapidwgs.sh .
```
3. Récupérer le mot de passe dans `~ya0902du/bin/get_data_rapidwgs.sh` pour se connecter à l'interface web de récupération des données. Choisir le dernier set de données.

4. Créer un fichier `file.list` et copier le contenu de la liste du lien du CNRGH dans ce fichier.
Tout ce que l'on veut c'est le nom du fichier, faire un `sed`.

5. Changer le chemin dans le fichier `get_data_rapidwgs.sh` et le lancer. Il va se connecter et télécharger les fichiers dans le répertoire courant.

6.
```sh
cp ~ya0902du/bin/perform_md5sum_check.sh .
./perform_md5sum_check.sh
```
Ce script crée un fichier failed ou ok.

7. Concaténer les fichiers read 1 et read 2 respectivement.
```sh
cp ~ya0902du/bin/auto_concat_cng.sh .
./auto_concat_cng.sh
```
8. Renommer les fichiers. Il n'y a pas de script prévu, le faire à la main. La correspondance se trouve dans la table suivi exome dans LabKey. `COOO1CPO.R1.fastq.gz` --> `dijen455.R1.fastq.gz`

9. Lancer l'analyse bioinformatique. Deux sortes d'analyse :
- génome complet
- génome fast :
```sh
qsub -v ANALYSISDIR=`pwd`,CONFIGFILE=/work/gad/shared/pipeline/2.5.0/common/analysis_config.tsv /work/gad/shared/pipeline/2.5.0/wgs_pipeline/full_fast_genome.sh
```

10. Mettre à jour LabKey de la même façon que pour les données d'Integragen. Dans le cas des génomes Fastgen RAPIDWG : lecteur 1 : Antonio Vitobello, lecteur 2 : Anne-Sophie Denommé-Pichon, lecteur 3 : Patrick Callier.

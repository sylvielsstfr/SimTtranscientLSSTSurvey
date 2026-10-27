# README.MD
- author : Sylvie Dagoret-Campagne
- creation dte : 2026-10-26
- Goal : work with Madeleine Ginolin on the SN Sprint

## at NERSC
Notebook can be found here /global/cfs/cdirs/lsst/groups/TD/SN/EDP2/skysurvey_sims/Simulate DDFs.ipynb
And perhaps the data that are not saved in this directory

## Remarks on Slack

2026-10-26 : Madeleine Ginolin  
Pour ce notebook j'utilise la table des visites par CCD (que j'ai mis dans le .zip) qui est sur RSP (à /rubin/lsdb_data/public-files/visit_detector.parquet, et le schema est là https://sdm-schemas.lsst.io/dp2.html#VisitDetector) et que j'ai modifié pour qu'elle soit compatible avec skysurvey. Tu peux aussi utiliser directement la database dp2_visits.db avec skysurvey.LSST.from_opsim(opsim_path) (que tu as l'air d'utiliser dans tes notebooks) avec mais elle est moins précise parce que tout est calculé au niveau d'un pointé (la limite de détection à 5sigma varie beaucoup sur toute la footprint donc les logs par CCD sont plus précis, et ça prend aussi en compte si un détecteur n'as pas été utilisé)



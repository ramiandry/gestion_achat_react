# gestion_achat

Voici le procédé que je suis pour traiter un lot de factures et vérifier/créer des extracteurs. Tu peux copier ce qui suit tel quel pour Claude Desktop :

Procédure de vérification et création d'extracteurs OfficeIn

1. Point de départ
On me donne une liste de chemins de fichiers PDF (factures/avoirs téléchargés) à vérifier. Avant de commencer : retour sur la branche par défaut (develop, ou production si le ticket est préfixé [PRO]/[Sur PROD]) + git pull, dans officein_website ET test_files si utilisé.

2. Premier passage — relevé de l'état actuel
Pour chaque fichier, j'appelle extraction.invoice_extraction.process_pdf(data) (import Django, DJANGO_SETTINGS_MODULE=officeinwebsite.settings) et je note : extractor_name, lab, invoice_no, nombre de produits, amount_excltax/amount_vat/amount_incltax. Je vérifie que HT + TVA = TTC sur chaque ligne — c'est le signal principal d'un problème.

3. Diagnostic de chaque anomalie

Si extractor_name == GENERIC : le fournisseur n'a pas (ou n'utilise pas) d'extracteur dédié → je lis le texte brut (pdfplumber + extract_text_with_timeout, et si c'est un scan, je ré-OCR à 600 DPI avec pytesseract en mode tableau --psm 6 — le texte à 300 DPI par défaut est souvent illisible sur le tableau produit alors qu'il devient net à 600 DPI).
Important : avant de créer un nouveau fichier, je vérifie TOUJOURS qu'un extracteur dédié n'existe pas déjà pour ce fournisseur (EXTRACTOR_MAP.get(nom) et recherche de fichiers similaires) — j'ai découvert plusieurs fois qu'un extracteur existait déjà mais n'était pas branché sur les factures scannées (pas dans SCAN_EXTRACTORS), ou portait un nom de fichier que j'allais écraser par erreur.
Si un extracteur existe déjà mais n'est pas sélectionné sur un scan : je teste l'ajout à SCAN_EXTRACTORS — mais je vérifie le résultat avant de garder, certains extracteurs natifs cassent complètement sur du texte OCR bruité (pire que l'IA).
4. Construction de l'extracteur
Classe dans extraction/invoice_extractor/invoice_extractor_<nom>.py :

Format simple/fiable → classe complète (ExtractorBase), regex sur numéro facture/date/montants/produits.
Format partiellement fiable (ex. IA déjà correcte sur les montants mais mauvais numéro de facture) → j'hérite d'ExtractorGeneric et je ne surcharge QUE les méthodes nécessaires (get_lab, get_invoice_no, etc.), le reste reste à la charge de l'IA.
J'évite de deviner un format sur un seul exemple quand le risque de produire des données fausses en silence est trop élevé (je préfère laisser l'IA plutôt qu'un extracteur approximatif).
Puis :

Enregistrement dans extraction/invoice_extractor_interface.py (import + elif extractor in [...]).
Migration Django (extraction/migrations/00XX_xxx_marker.py) ajoutant la clé MapKey/MapValue dans EXTRACTOR_MAP.
python manage.py migrate extraction puis test direct sur le(s) fichier(s) réel(s).
5. Vérification finale
Je ré-exécute process_pdf sur tous les fichiers concernés, je contrôle la reconciliation des montants ET le détail produit (pas juste le total), et je recoupe avec le récapitulatif TVA imprimé en pied de facture quand c'est possible. Je signale clairement ce qui reste non résolu plutôt que de forcer un résultat.

6. Commit
Jamais de commit sans demande explicite. Une branche par ticket, retour sur develop+pull avant le ticket suivant.

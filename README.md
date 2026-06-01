# Laboratoire d'Audit de Sécurité Android avec Drozer

## Présentation du Projet

Ce laboratoire documente une analyse de sécurité offensive réalisée sur l'application Android DIVA (Damn Insecure and Vulnerable App). L'objectif est d'identifier les composants exposés, d'évaluer les risques de fuite de données et de proposer des mesures de remédiation conformes aux standards OWASP MASVS.

| Champ | Valeur |
|---|---|
| **Auditrice** | Nisrine Gorfti |
| **Établissement** | EMSI |
| **Cible** | `jakhar.aseem.diva` |
| **Outils** | Drozer, ADB, Émulateur Android (API 28) |
| **Date** | 2026-06-01 |

---

## Étape 1 : Configuration de l'Environnement

L'infrastructure de test a été mise en place pour permettre la communication entre la machine hôte et l'agent Drozer sur l'émulateur.

- Lancement de l'émulateur via Android Studio
- Installation de l'Agent Drozer et de l'application cible :

```bash
adb install drozer-agent.apk
adb install diva.apk
```

- Activation du serveur sur l'application Drozer Agent (Port 31415)
- Configuration du port forwarding :

```bash
adb forward tcp:31415 tcp:31415
```

![Configuration de l'environnement](https://github.com/user-attachments/assets/3b391f64-b288-498d-bc84-68b64864c025)

---

## Étape 2 : Connexion et Validation

La connexion a été établie avec succès entre la console hôte et l'appareil.

- Commande de connexion : `drozer console connect`
- Vérification de la cible :

```bash
dz> run app.package.info -a jakhar.aseem.diva
```

Résultat : Package détecté, version 1.0, permissions `WRITE/READ_EXTERNAL_STORAGE` identifiées.

![Connexion Drozer établie](https://github.com/user-attachments/assets/c49b7fdd-5410-49a6-8d1f-473e8fb1583a)

---

## Étape 3 : Cartographie des Composants (Surface d'Attaque)

L'analyse de la surface d'attaque a révélé plusieurs composants critiques exposés sans aucune permission (`null`).

![Cartographie des composants](https://github.com/user-attachments/assets/861afef8-1c00-4a23-9f7e-7502e8cae0ab)

### Tableau récapitulatif des composants

| Type | Nom du composant | Exporté | Protection |
|---|---|---|---|
| Activity | `jakhar.aseem.diva.MainActivity` | Oui | Aucune (null) |
| Activity | `jakhar.aseem.diva.APICredsActivity` | Oui | Aucune (null) |
| Activity | `jakhar.aseem.diva.APICreds2Activity` | Oui | Aucune (null) |
| Provider | `jakhar.aseem.diva.NotesProvider` | Oui | Aucune (null) |

---

## Étape 4 : Vérification des Protections & Scan URI

Scan des chemins de données (URIs) accessibles pour le Content Provider afin de tester la confidentialité des données.

```bash
dz> run scanner.provider.finduris -a jakhar.aseem.diva
```

Résultat : Les URIs suivantes sont accessibles en lecture libre (sans authentification) :
```
content://jakhar.aseem.diva.provider.notesprovider/notes
```

![Scan des URIs](https://github.com/user-attachments/assets/a04be59a-0710-4982-bf88-312880a8acbc)

---

## Étape 5 : Analyse des Risques & Triage

Les vulnérabilités ont été classées par sévérité selon leur impact potentiel sur l'utilisateur final.

![Triage des vulnérabilités](https://github.com/user-attachments/assets/f0db9175-52e0-4fe6-92c5-8be1341159be)

| ID | Composant | Vulnérabilité | Confiance | Sévérité | Impact | Statut |
|---|---|---|---|---|---|---|
| V1 | NotesProvider | URIs accessibles sans permission | Élevée | 🔴 Critique | Fuite totale des données personnelles | À corriger |
| V2 | APICredsActivity | Activité exportée (Intent Filter) | Élevée | 🔴 Élevée | Vol d'identifiants API sensibles | À corriger |
| V3 | AndroidManifest | Mode `debuggable="true"` activé | Élevée | 🟡 Moyenne | Reverse engineering facilité | À corriger |
| V4 | MainActivity | Composant exporté (Standard) | Élevée | 🔵 Faible | Surface d'attaque de base | Accepté |

---

## Étape 6 : Mapping OWASP MASVS

Alignement des découvertes avec le référentiel de sécurité mobile de l'OWASP.

![Mapping OWASP](https://github.com/user-attachments/assets/36b17096-47e2-4542-aad6-73e76babb252)

| ID | Vulnérabilité | Référence MASVS | Description |
|---|---|---|---|
| V1 | Activités exportées | MSTG-PLATFORM-1 | Exposition de composants inutiles |
| V2 | Provider non protégé | MSTG-STORAGE-2 | Stockage de données sensibles non sécurisé |
| V3 | Mode Debug activé | MSTG-CODE-2 | Code de débogage présent en production |

---

## Étape 7 : Remédiations Proposées

### 1. Sécurisation des Activities

```xml
<activity 
    android:name=".APICredsActivity" 
    android:exported="false" />
```

### 2. Protection du Content Provider

```xml
<provider
    android:name=".NotesProvider"
    android:authorities="jakhar.aseem.diva.provider.notesprovider"
    android:exported="false" />
```

### 3. Durcissement du Manifeste

```xml
<application 
    android:debuggable="false" 
    android:allowBackup="false"
    android:icon="@mipmap/ic_launcher"
    android:label="@string/app_name">
</application>
```

---

## Constats Majeurs

1. **URIs accessibles sans permission** — Fuite totale des données personnelles via le NotesProvider
2. **Activités exportées sans protection** — Vol d'identifiants API possible via Intent externe
3. **Mode Debug activé** — Facilite le reverse engineering et l'attachement d'un débogueur JDWP
4. **Backup ADB autorisé** — Extraction physique des données applicatives possible
5. **Permissions de stockage excessives** — Accès carte SD non justifié


# LAB 19 : Snake –PwnSec CTF 2024 Mobile Hard

## Objectif du challenge

L’application Android `Snake.apk` implémente plusieurs mécanismes de protection anti‑reverse :

- Détection de **root**
- Détection d’**émulateur**
- Détection de **Frida** via une librairie native

Elle lit un fichier YAML depuis le stockage externe et le parse à l’aide d’une version vulnérable de **SnakeYAML** (probablement la 1.33, vulnérable à **CVE-2022-1471**).

L’objectif est d’exploiter cette vulnérabilité de désérialisation unsafe pour instancier une classe cachée nommée `BigBoss`.  
Cette classe charge une librairie native et, lorsqu’elle reçoit la bonne chaîne en paramètre, appelle une fonction JNI qui génère et affiche le flag dans les logs Android (`logcat`).

> **Contrairement à d’autres challenges, Frida ne peut pas être utilisé ici à cause des détections natives. La solution repose sur du patching statique de l’APK + une payload de désérialisation YAML.**
  
- **Techniques principales** :
  - Analyse statique approfondie (Jadx + apktool)
  - Patching Smali pour bypasser les anti‑debug / anti‑root
  - Exploitation de désérialisation SnakeYAML (CVE‑2022‑1471)
  - Utilisation d’Intent via ADB
  - Récupération du flag via logcat

---

## Flag attendu

```
PWNSEC{W3'r3_N0t_T00l5_0f_The_g0v3rnm3n7_0R_4ny0n3_3ls3}
```

---

## Prérequis outils

| Outil | Utilité |
|-------|---------|
| [Jadx-GUI](https://github.com/skylot/jadx) | Analyse Java décompilée |
| [apktool](https://apktool.org/) | Décompilation / recompilation Smali |
| [uber-apk-signer](https://github.com/patrickfav/uber-apk-signer) ou `apksigner` | Signature de l’APK patché |
| ADB | Communication avec l’émulateur / appareil |
| Émulateur Android (API ≤ 28 recommandé) | Évite certaines détections natives |

---

## Étapes d’exploitation

### 0. Préparation de l’environnement

```bash
adb install snake.apk
```

L’application se ferme immédiatement en présence de root, d’émulateur ou de Frida → un patching est obligatoire.

---

### 1. Analyse statique avec Jadx

Ouvrir `snake.apk` dans **Jadx-GUI** et identifier :

- **Package principal** : `com.pwnsec.snake`
- **Classe d’entrée** : `MainActivity`

Dans `MainActivity` (souvent dans `onCreate`), on trouve la logique suivante :

- Vérification d’un extra Intent nommé `SNAKE` avec la valeur `BigBoss`
- Si OK → accès au stockage externe (`/sdcard/Snake/Skull_Face.yml`)
- Lecture et parsing YAML avec `SnakeYAML`

**Classe cible** : `com.pwnsec.snake.BigBoss`

- Charge une librairie native (`System.loadLibrary(...)`)
- Contient une méthode (ex: `Bigboss(String)`) qui attend la chaîne exacte `Snaaaaaaaaaaaaaake`
- Appelle alors une fonction native `stringFromJNI()` qui logue le flag

**Protections détectées** (à bypasser) :

- Root : `Build.TAGS`, présence de `su`, `/system/app/Superuser.apk`
- Émulateur : propriétés `ro.hardware`, `ro.product.model`
- Frida : checks natifs sur processus / ports

---

### 2. Patching Smali (anti‑root / anti‑émulateur / anti‑Frida)

#### Décompilation

```bash
apktool d snake.apk -o snake_smali
cd snake_smali/smali/com/pwnsec/snake/
```

#### Modification typique

Dans les méthodes de détection (repérées via Jadx avec des strings comme `"root"`, `"su"`, `"emulator"`, `"frida"`) :

- Une instruction `if-nez` ou `if-eqz` décide de l’arrêt de l’app
- Pour **bypasser** :
  - Remplacer la condition par un `goto` vers la partie “safe”
  - Ou forcer la fonction à retourner `false` (`const/4 v0, 0x0` / `return v0`)

#### Recompilation & signature

```bash
apktool b snake_smali -o snake_patched.apk

# Signature (exemple avec uber-apk-signer)
java -jar uber-apk-signer.jar -a snake_patched.apk

# Installation
adb install -r snake_patched.apk
```
---

### 3. Création du payload YAML (exploit CVE‑2022‑1471)

#### Création du dossier et du fichier sur l’appareil

```bash
adb shell mkdir -p /sdcard/Snake
```

#### Contenu du fichier `Skull_Face.yml`

```yaml
!!com.pwnsec.snake.BigBoss ["Snaaaaaaaaaaaaaake"]
```

**Explication** :

- `!!com.pwnsec.snake.BigBoss` : tag YAML forçant l’instanciation directe de la classe Java (désérialisation unsafe)
- `["Snaaaaaaaaaaaaaake"]` : argument passé au constructeur / méthode, correspondant à la chaîne attendue par `BigBoss`

#### Envoi du fichier

```bash
adb push Skull_Face.yml /sdcard/Snake/Skull_Face.yml
```

---

### 4. Lancement de l’application avec l’Intent requis

```bash
adb shell am start -n com.pwnsec.snake/.MainActivity -e SNAKE BigBoss
```

L’extra `SNAKE` avec la valeur `BigBoss` déclenche la lecture du fichier YAML, la désérialisation malveillante, l’instanciation de `BigBoss`, puis l’appel natif qui logue le flag.

---

### 5. Récupération du flag dans logcat

```bash
adb logcat | grep -i "PWNSEC"
# ou
adb logcat | grep -i "snake"
```

Le flag apparaît dans les logs sous la forme attendue.

---
nge (simulé)** : CTF Android / Reverse Engineering  
**Licence du README** : Libre d’utilisation pour documentation technique ou éducative.
```

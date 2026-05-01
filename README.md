# Lab 13 — Bypass de détection root avec Objection & Frida

**Cours :** Sécurité mobile — Analyse dynamique Android  
**App cible :** RootBeer Sample (`com.scottyab.rootbeer.sample`)  
**Environnement :** Windows 11, Android 13 (émulateur AVD), Objection 1.12.4, Frida

---

## Table des matières

1. [Objectif du lab](#1-objectif-du-lab)
2. [Environnement & outils](#2-environnement--outils)
3. [Exercice 1 — Preuve d'installation et de connexion (20 pts)](#3-exercice-1--preuve-dinstallation-et-de-connexion-20-pts)
4. [Exercice 2 — Démarrage et visibilité (20 pts)](#4-exercice-2--démarrage-et-visibilité-20-pts)
5. [Exercice 3 — Bypass Java avec Objection (40 pts)](#5-exercice-3--bypass-java-avec-objection-40-pts)
6. [Exercice 4 — Bonus natif avec frida-trace (20 pts)](#6-exercice-4--bonus-natif-avec-frida-trace-20-pts)
7. [Résumé des commandes](#7-résumé-des-commandes)
8. [Conclusion](#8-conclusion)

---

## 1. Objectif du lab

Démontrer le contournement de la détection de root sur une application Android en utilisant **Objection** (CLI construite sur Frida). L'app cible, **RootBeer Sample**, effectue de nombreux checks Java et natifs pour détecter si le device est rooté. L'objectif est de neutraliser ces checks sans modifier l'APK.

---

## 2. Environnement & outils

| Outil | Version | Rôle |
|---|---|---|
| Python | 3.14.4 | Runtime pour pip/pipx |
| pipx | dernière | Isolation des outils CLI Python |
| Objection | 1.12.4 | Interface haut niveau sur Frida |
| Frida | — | Framework d'instrumentation dynamique |
| ADB | — | Communication avec l'émulateur Android |
| Android Emulator | Android 13 | Device cible (rooté via Magisk) |

---

## 3. Exercice 1 — Preuve d'installation et de connexion (20 pts)

### Installation d'Objection

Objection a été installé via `pipx` pour une isolation propre :

```powershell
pip install --user pipx
pipx ensurepath
pipx install objection
```

Sortie de l'installation :

```
WARNING: Skipping setuptools as it is not installed.
  installed package objection 1.12.4, installed using Python 3.14.4
  These apps are now available
    - objection.exe
done! ✨ 🌟 ✨
```

### Ajout du PATH et vérification

```powershell
$env:PATH += ";C:\Users\Admin\.local\bin"
objection --help
```

> **Note :** `objection --version` n'est pas une option valide en v1.12.4 ; la réponse à `--help` confirme l'installation.

### Vérification de Frida

```powershell
frida --version
frida-ps -U
```

Sortie partielle de `frida-ps -U` (device connecté) :

```
 PID  Name
----  -------------------------
4394  RootBeer Sample
 284  magiskd
 312  zygote64
...
```

### Vérification ADB

```powershell
adb devices
```

Sortie attendue :

```
List of devices attached
emulator-5554   device
```

---

## 4. Exercice 2 — Démarrage et visibilité (20 pts)

### Lancement de frida-server sur l'émulateur

```powershell
adb shell "/data/local/tmp/frida-server &"
```

Vérification que le processus tourne :

```powershell
adb shell ps -e | findstr frida
```

Sortie :

```
root  4911  1  12791928  136484  do_sys_poll  0 S  frida-server
```

### Lancement d'Objection sur l'app cible

```powershell
objection -n com.scottyab.rootbeer.sample start --startup-command "android root disable"
```

Invite obtenue :

```
com.scottyab.rootbeer.sample on (Android: 13) [usb] #
```

> **Note :** L'option `-g` et la commande `explore` sont dépréciées en v1.12.4. Utiliser `-n` et `start`.

---

## 5. Exercice 3 — Bypass Java avec Objection (40 pts)

### Avant le bypass

Sans instrumentation, l'app RootBeer Sample affiche **"ROOTED"** en rouge avec la majorité des checks marqués comme détectés.

### Commande exécutée

```
android root disable
```

Exécutée automatiquement via `--startup-command` au démarrage de la session.

### Logs Objection (hooks installés)

```
(agent) Registering job 336699. Name: root-detection-disable
(agent) [336699] RootBeer->detectRootCloakingApps() check detected, marking as false.
(agent) [336699] RootBeer->detectTestKeys() check detected, marking as false.
(agent) [336699] RootBeer->checkForBinary() check detected, marking as false.
(agent) [336699] RootBeer->checkForBinary() check detected, marking as false.
(agent) [336699] RootBeer->checkSuExists() check detected, marking as false.
(agent) [336699] RootBeer->checkForDangerousProps() check detected, marking as false.
(agent) [336699] RootBeerNative->checkForRoot() check detected, marking as 0.
(agent) [336699] RootBeer->checkForBinary() check detected, marking as false.
```

### Résultat après bypass

| Check | Avant | Après |
|---|---|---|
| Root Management Apps | Détecté | Neutralisé |
| Potentially Dangerous Apps | Détecté | Neutralisé |
| Root Cloaking Apps | Détecté | Neutralisé |
| TestKeys | Détecté | Neutralisé |
| BusyBox Binary | Détecté | Neutralisé |
| SU Binary | Détecté | Neutralisé |
| 2nd SU Binary check | Détecté | Neutralisé |
| For RW Paths | Détecté | **Partiellement** (voir Exercice 4) |
| Dangerous Props | Détecté | Neutralisé |
| Root via native check | Détecté | Neutralisé (`checkForRoot` → 0) |
| SE linux Flag Is Enabled | Détecté | Neutralisé |
| Magisk specific checks | Détecté | Neutralisé |

### Ce que fait `android root disable`

Objection installe des hooks Java via Frida qui :

- Forcent `android.os.Build.TAGS` à renvoyer `release-keys`
- Font échouer `java.io.File.exists()` sur des chemins suspects (`/system/xbin/su`, `busybox`…)
- Neutralisent `Runtime.getRuntime().exec("su")`
- Désactivent des méthodes de libs connues (RootBeer `isRooted()`, etc.)

---

## 6. Exercice 4 — Bonus natif avec frida-trace (20 pts)

### Contexte

Le check **"For RW Paths"** correspond à `checkForRWPaths()` dans RootBeer. Il lit `/proc/mounts` via des appels système natifs (`open`, `read`) pour vérifier si `/system` est monté en écriture. Objection ne le couvre pas nativement.

### Identification avec frida-trace

```powershell
frida-trace -U -i open -i access -i stat -i openat com.scottyab.rootbeer.sample
```

Cette commande trace tous les appels aux fonctions natives `open`, `access`, `stat`, `openat` et permet d'identifier ceux qui lisent `/proc/mounts`.

### Observation de la méthode avant le hook

Avant de forcer le retour, on observe d'abord ce que retourne la méthode en conditions réelles :

```
android hooking watch class_method com.scottyab.rootbeer.RootBeer.checkForRWPaths --dump-return
```

Cette commande installe un hook passif qui :
- intercepte chaque appel à `checkForRWPaths()`
- affiche les arguments passés (s'il y en a)
- affiche la valeur de retour réelle (`true` → path RW détecté)

Exemple de sortie dans la console Objection :

```
(agent) Watching com.scottyab.rootbeer.RootBeer.checkForRWPaths
(agent) [calls] com.scottyab.rootbeer.RootBeer.checkForRWPaths()
(agent) [retn] com.scottyab.rootbeer.RootBeer.checkForRWPaths() --> true
```

Cela confirme que la méthode retourne `true` (root détecté) et qu'il faut la neutraliser.

### Hook Java de la méthode passerelle

Une fois le comportement confirmé, on force le retour à `false` :

```
android hooking set return_value com.scottyab.rootbeer.RootBeer.checkForRWPaths false
```

### Script Frida alternatif

```javascript
Java.perform(function () {
  var RootBeer = Java.use('com.scottyab.rootbeer.RootBeer');
  RootBeer.checkForRWPaths.implementation = function () {
    console.log('[*] checkForRWPaths hooked → returning false');
    return false;
  };
});
```

Injection :

```powershell
frida -U -n com.scottyab.rootbeer.sample -e "Java.perform(function(){ var RootBeer = Java.use('com.scottyab.rootbeer.RootBeer'); RootBeer.checkForRWPaths.implementation = function(){ return false; }; });"
```

---

## 7. Résumé des commandes

```powershell
# Installation
pip install --user pipx
pipx install objection

# Démarrer frida-server sur l'émulateur
adb shell "/data/local/tmp/frida-server &"

# Vérifier la connexion
frida-ps -U
adb devices

# Lancer Objection avec bypass root
objection -n com.scottyab.rootbeer.sample start --startup-command "android root disable"

# Dans la console Objection — explorer les classes
android hooking search classes root
android hooking search methods isRooted

# Forcer le retour d'une méthode spécifique
android hooking set return_value com.scottyab.rootbeer.RootBeer.checkForRWPaths false

# Tracer les appels natifs
frida-trace -U -i open -i access -i stat -i openat com.scottyab.rootbeer.sample

# Automatiser plusieurs actions au démarrage
objection -n com.scottyab.rootbeer.sample start \
  --startup-command "android root disable" \
  --startup-command "android sslpinning disable"
```

---

## 8. Conclusion

Ce lab démontre l'efficacité d'Objection pour contourner les mécanismes de détection root côté Java sur Android. En une seule commande (`android root disable`), la majorité des checks RootBeer sont neutralisés sans modifier l'APK ni nécessiter de compétences Frida avancées.

Les limites apparaissent sur les checks purement natifs (C/C++) comme `checkForRWPaths`, qui nécessitent soit un hook au niveau de la méthode Java passerelle, soit un script Frida ciblant directement les syscalls (`open`, `stat`, `access`).

**Points clés retenus :**
- Objection est une surcouche Frida qui simplifie l'instrumentation dynamique Android
- Les hooks Java couvrent l'essentiel des détections courantes
- `frida-trace` est indispensable pour identifier les appels natifs suspects
- Les checks natifs nécessitent des scripts Frida custom pour un bypass complet

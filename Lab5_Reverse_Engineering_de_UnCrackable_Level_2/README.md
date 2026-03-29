# LAB 5 — Reverse Engineering de UnCrackable Level 2

> **Cours :** Sécurité des applications mobiles — MLIAEdu
> **Date :** 23 Mars 2026
> **Analyste :** Adansar Abdessamad
> **APK analysé :** UnCrackable-Level2.apk (OWASP MSTG)

---

## 🗺️ Table des matières

1. [Vue d'ensemble](#vue-densemble)
2. [Outils utilisés](#outils-utilisés)
3. [Partie 1 — Découverte de l'application](#partie-1--découverte-de-lapplication)
4. [Partie 2 — Analyse Java avec JADX](#partie-2--analyse-java-avec-jadx)
5. [Partie 3 — Extraction de la bibliothèque native](#partie-3--extraction-de-la-bibliothèque-native)
6. [Partie 4 — Analyse native avec Ghidra](#partie-4--analyse-native-avec-ghidra)
7. [Partie 5 — Décodage du secret](#partie-5--décodage-du-secret)
8. [Partie 6 — Validation](#partie-6--validation)
9. [Résumé du flux](#résumé-du-flux)
10. [Ce qu'il faut retenir](#ce-quil-faut-retenir)

---

## Vue d'ensemble

Ce lab couvre le reverse engineering d'une application Android qui cache sa logique de vérification dans une bibliothèque native (code C compilé). L'APK utilisé est **UnCrackable Level 2** de l'OWASP Mobile Security Testing Guide (MSTG), une application intentionnellement vulnérable conçue à des fins pédagogiques.

L'objectif est de retrouver la chaîne secrète attendue par l'application en suivant le chemin de la donnée depuis l'interface utilisateur jusqu'au code natif.

---

## Outils utilisés

| Outil | Version | Usage |
|---|---|---|
| adb | Android SDK | Installation et interaction avec l'émulateur |
| JADX GUI | latest | Décompilation APK + analyse Java |
| Ghidra | 12.0.4 | Analyse du code natif (libfoo.so) |
| PowerShell | Windows | Extraction du contenu de l'APK |
| Android Emulator | API 29 AOSP | Test de l'application |

---

## Partie 1 — Découverte de l'application

### Étape 1 — Installation de l'APK

```bash
adb devices
adb install UnCrackable-Level2.apk
```

📸 **Screenshots :**
![adb install](images/02_adb_install.png)

---

### Étape 2 — Lancement et observation

Au lancement, l'application affiche immédiatement une alerte **"Root detected!"** et se ferme automatiquement. Cela indique que l'application implémente une **protection anti-tampering** dès le démarrage.

📸 **Screenshot :**
![Root Detected](images/01_root_detected.png)

**Observation :** L'analyse dynamique est bloquée par la protection anti-root. L'app se ferme même sur un émulateur AOSP non rooté (API 29). On procède donc uniquement par **analyse statique**.

---

## Partie 2 — Analyse Java avec JADX

### Étape 3 — Ouverture de l'APK dans JADX GUI

Lancement de JADX GUI → File → Open file → UnCrackable-Level2.apk

📸 **Screenshot :**
![JADX Manifest](images/04_jadx_manifest.png)

---

### Étape 4 — Analyse de MainActivity

Navigation vers :
```
Source code → sg.vantagepoint.uncrackable2 → MainActivity
```

**Découvertes importantes dans MainActivity :**

```java
public class MainActivity extends c {

    private CodeCheck m;

    static {
        System.loadLibrary("foo");  // Chargement de la bibliothèque native !
    }

    private native void init();    // Méthode native dans MainActivity aussi !

    protected void onCreate(Bundle bundle) {
        init();
        if (b.a() || b.b() || b.c()) {
            a("Root detected!");       // Protection anti-root
        }
        if (a.a(getApplicationContext())) {
            a("App is debuggable!");   // Protection anti-debug
        }
        // Thread qui surveille en continu la connexion d'un debugger
        new AsyncTask<Void, String, String>() {
            public String doInBackground(Void... voidArr) {
                while (!Debug.isDebuggerConnected()) {
                    SystemClock.sleep(100L);
                }
                return null;
            }
            public void onPostExecute(String str) {
                MainActivity.this.a("Debugger detected!");
            }
        }.execute(null, null, null);

        this.m = new CodeCheck();
    }

    public void verify(View view) {
        String string = ((EditText) findViewById(R.id.edit_text)).getText().toString();
        if (this.m.a(string)) {        // Vérification déléguée à CodeCheck !
            alertDialogCreate.setTitle("Success!");
        } else {
            alertDialogCreate.setTitle("Nope...");
        }
    }
}
```

**🚨 Protections identifiées :**

| Protection | Méthode | Impact |
|---|---|---|
| Anti-root | `b.a()`, `b.b()`, `b.c()` | Ferme l'app si root détecté |
| Anti-debug | `a.a()` | Ferme l'app si debuggable |
| Anti-debugger dynamique | `AsyncTask` + `Debug.isDebuggerConnected()` | Thread continu qui surveille |
| Native init | `init()` native | Initialisation native au démarrage |

📸 **Screenshot :**
![MainActivity](images/05_mainactivity.png)

---

### Étape 5 — Analyse de la classe CodeCheck

Navigation vers :
```
Source code → sg.vantagepoint.uncrackable2 → CodeCheck
```

```java
public class CodeCheck {
    private native boolean bar(byte[] bArr);  // Méthode native !

    public boolean a(String str) {
        return bar(str.getBytes());  // Convertit String en bytes et appelle bar()
    }
}
```

**Observations :**
- La méthode `bar()` est déclarée `native` → son code est dans `libfoo.so`
- La vérification réelle se fait dans le code natif, pas en Java
- Il faut donc analyser `libfoo.so` pour trouver le secret

📸 **Screenshot :**
![CodeCheck](images/06_codecheck.png)

---

## Partie 3 — Extraction de la bibliothèque native

### Étape 6 — Extraction du contenu de l'APK

```powershell
Add-Type -Assembly System.IO.Compression.FileSystem
$zip = [System.IO.Compression.ZipFile]::OpenRead("$PWD\UnCrackable-Level2.apk")
$zip.Entries | ForEach-Object {
    $outPath = "C:\Users\HP\Desktop\APK-Analysis\uncrackable_l2\" + $_.FullName
    $outDir = Split-Path $outPath -Parent
    if (!(Test-Path $outDir)) { New-Item -ItemType Directory -Path $outDir | Out-Null }
    if (!$_.FullName.EndsWith("/")) {
        [System.IO.Compression.ZipFileExtensions]::ExtractToFile($_, $outPath, $true)
    }
}
$zip.Dispose()
```

### Étape 7 — Localisation de libfoo.so

```powershell
Get-ChildItem -Recurse "C:\Users\HP\Desktop\APK-Analysis\uncrackable_l2\lib"
```

**Résultat :**
```
lib/
  arm64-v8a/libfoo.so    (14176 bytes)
  armeabi-v7a/libfoo.so  (13948 bytes)
  x86/libfoo.so          (13788 bytes)
  x86_64/libfoo.so       (14440 bytes)
```

L'APK contient 4 versions de `libfoo.so` pour chaque architecture. On utilise `x86/libfoo.so` pour l'analyse dans Ghidra.

📸 **Screenshots :**
![libfoo extracted](images/07_libfoo_extracted.png)
![libfoo extracted full](images/07b_libfoo_extracted_full.png)

---

## Partie 4 — Analyse native avec Ghidra

### Étape 8 — Import de libfoo.so dans Ghidra

Lancement de Ghidra :
```powershell
cd C:\Users\HP\Downloads\ghidra_12.0.4_PUBLIC_20260303\ghidra_12.0.4_PUBLIC
set JAVA_HOME=C:\Users\HP\AppData\Local\Programs\Microsoft\jdk-21.0.10.7-hotspot
set PATH=%JAVA_HOME%\bin;%PATH%
support\launch.bat fg jdk Ghidra 4G "" ghidra.GhidraRun
```

Création du projet : `File → New Project → UnCrackable2`

Import du fichier : `File → Import File → x86/libfoo.so`

**Informations d'import :**

| Champ | Valeur |
|---|---|
| Language | x86:LE:32:default |
| Compiler | gcc |
| Format | ELF (shared object) |
| # Functions | 26 |
| # Symbols | 46 |
| SHA256 | 7119c6b8bf6019bd2011ebd1ac9e69f7d71e08c9c41d84abd8a8d438712b0548 |

📸 **Screenshot :**
![Ghidra Import](images/08_ghidra_import.png)

---

### Étape 9 — Analyse automatique

Après import : double-clic sur `libfoo.so` → **"Analyze"** → **"Yes"** → **"Analyze"**

📸 **Screenshot :**
![Ghidra Analysis](images/09_ghidra_analysis.png)

---

### Étape 10 — Identification de la fonction JNI

Dans **Symbol Tree → Exports**, on identifie la fonction JNI :
```
Java_sg_vantagepoint_uncrackable2_CodeCheck_bar
```

Cette fonction est la version native de `CodeCheck.bar()` côté Java.

---

### Étape 11 — Lecture du pseudo-code et découverte du secret

Pseudo-code décompilé par Ghidra :

```c
undefined4 Java_sg_vantagepoint_uncrackable2_CodeCheck_bar
    (int *param_1, undefined4 param_2, undefined4 param_3)
{
    char *__s1;
    int iVar1;
    undefined4 uVar2;
    int in_GS_OFFSET;
    char local_30 [24];
    int local_18;

    local_18 = *(int *)(in_GS_OFFSET + 0x14);
    if (DAT_00014008 == '\x01') {
        builtin_strncpy(local_30, "Thanks for all the fish", 0x18);  // SECRET !
        __s1 = (char *)(**(code **)(*param_1 + 0x2e0))(param_1, param_3, 0);
        iVar1 = (**(code **)(*param_1 + 0x2ac))(param_1, param_3);
        if ((iVar1 == 0x17) && (iVar1 = strncmp(__s1, local_30, 0x17), iVar1 == 0)) {
            uVar2 = 1;
            goto LAB_00011009;
        }
    }
    uVar2 = 0;
}
```

**🎯 Secret trouvé directement en clair :**
```c
builtin_strncpy(local_30, "Thanks for all the fish", 0x18);
```

📸 **Screenshot :**
![Ghidra Secret](images/10_ghidra_secret.png)

---

## Partie 5 — Décodage du secret

Dans ce cas, le secret était **directement visible en clair** dans le pseudo-code Ghidra. Aucun décodage supplémentaire n'était nécessaire.

Si la chaîne avait été stockée en hexadécimal, le décodage aurait été :

```python
hex_data = "6873696620656874206c6c6120726f6620736b6e616854"
decoded = bytes.fromhex(hex_data).decode('ascii')
secret = decoded[::-1]  # Inversion
print(secret)  # Thanks for all the fish
```

**Secret final :**
```
Thanks for all the fish
```

---

## Partie 6 — Validation

**Note :** La validation dans l'application n'a pas pu être effectuée car l'application implémente des protections anti-émulateur qui ferment l'app même sur un émulateur AOSP non rooté (API 29). Cela démontre la robustesse des mécanismes anti-tampering de l'application.

Le secret `Thanks for all the fish` a été confirmé par l'analyse statique du code natif via Ghidra, conformément au write-up de référence OWASP MSTG.

---

## Résumé du flux

```
Utilisateur
    ↓
MainActivity.verify()
    ↓
CodeCheck.a(string)
    ↓
CodeCheck.bar(str.getBytes())  ← méthode native
    ↓
libfoo.so → Java_sg_vantagepoint_uncrackable2_CodeCheck_bar()
    ↓
strncmp(input, "Thanks for all the fish", 0x17)
    ↓
succès / échec
```

---

## Ce qu'il faut retenir

Ce lab illustre un concept fondamental en sécurité mobile : **une logique sensible peut être déplacée du code Java vers le code natif** pour compliquer l'analyse. La décompilation Java seule est insuffisante — il faut utiliser un outil comme Ghidra pour analyser les bibliothèques natives.

**Vulnérabilités identifiées :**

| # | Titre | Sévérité | Localisation |
|---|---|---|---|
| 1 | Secret hardcodé en clair dans le code natif | 🔴 Élevée | libfoo.so |
| 2 | Protection anti-root contournable par analyse statique | 🟠 Moyenne | MainActivity |
| 3 | Protection anti-debug contournable par analyse statique | 🟠 Moyenne | MainActivity |
| 4 | Pas de chiffrement de la chaîne secrète | 🔴 Élevée | libfoo.so |

**Checklist finale :**

- [x] APK installé et comportement observé
- [x] Protection anti-root identifiée et documentée
- [x] MainActivity analysée avec JADX
- [x] Classe CodeCheck analysée — lien JNI identifié
- [x] libfoo.so extrait de l'APK (4 architectures)
- [x] libfoo.so importé et analysé dans Ghidra
- [x] Fonction JNI `CodeCheck_bar` localisée dans Exports
- [x] Secret `Thanks for all the fish` trouvé en clair
- [x] Flux complet de l'application documenté

---

*© 2026 MLIAEdu Platform — LAB 5 Reverse Engineering UnCrackable Level 2*

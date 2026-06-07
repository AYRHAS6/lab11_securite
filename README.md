# Lab 11 — Bypass de détection Root avec Frida sur RootBeer Sample

## Environnement

| Composant | Détail |
|-----------|--------|
| OS analyste | Windows 11 |
| Frida Client | frida-tools 14.8.1 / frida 17.9.1 |
| Émulateur | Genymotion — Pixel 3 (Android x86) |
| Frida Server | frida-server-17.9.1-android-x86 |
| Application cible | RootBeer Sample — `com.scottyab.rootbeer.sample` |

<img width="379" height="51" alt="4" src="https://github.com/user-attachments/assets/a9ddebdf-ae73-493f-b78a-9ab0c3295249" />


## Qu'est-ce que RootBeer ?

RootBeer est une bibliothèque Android open source permettant aux développeurs de détecter si un appareil est rooté. L'application RootBeer Sample expose visuellement le résultat de chaque vérification sous forme de liste : une coche verte signifie que le check est passé (non rooté), une croix rouge signifie que le root a été détecté.

Les vérifications effectuées couvrent la présence de binaires su et BusyBox, les propriétés système dangereuses, les applications de gestion de root (Magisk, SuperSU), les chemins accessibles en écriture, les flags SELinux, et les checks natifs bas niveau.

---

## Le script bypass_rootbeer.js — Explication globale

Le script suivant utilise l'API Java de Frida pour intercepter et neutraliser **chaque méthode de détection** de la bibliothèque RootBeer, une par une. Toutes les méthodes sont surchargées pour retourner systématiquement `false`, ce qui trompe l'application en lui faisant croire que l'appareil n'est pas rooté.

```javascript
Java.perform(function(){
    var RootBeer = Java.use("com.scottyab.rootbeer.RootBeer");
    var Utils = Java.use("com.scottyab.rootbeer.util.Utils");   
    
    RootBeer.detectRootManagementApps.overload().implementation = function(){
        return false;
    };
    
    RootBeer.detectPotentiallyDangerousApps.overload().implementation = function(){
        return false;
    };
    
    RootBeer.detectTestKeys.overload().implementation = function(){
        return false;
    };
    
    RootBeer.checkForBusyBoxBinary.overload().implementation = function(){
        return false;
    };
    
    RootBeer.checkForSuBinary.overload().implementation = function(){
        return false;
    };
    
    RootBeer.checkSuExists.overload().implementation = function(){
        return false;
    };
    
    RootBeer.checkForRWPaths.overload().implementation = function(){
        return false;
    };
    
    RootBeer.checkForDangerousProps.overload().implementation = function(){
        return false;
    };
    
    RootBeer.checkForRootNative.overload().implementation = function(){
        return false;
    };
    
    RootBeer.detectRootCloakingApps.overload().implementation = function(){
        return false;
    };
    
    Utils.isSelinuxFlagInEnabled.overload().implementation = function(){
        return false;
    };
    
    RootBeer.checkForMagiskBinary.overload().implementation = function(){
        return false;
    };
    
    RootBeer.isRooted.overload().implementation = function(){
        return false;
    };
});
```

### Ce que fait chaque hook

| Méthode hookée | Rôle original | Valeur retournée |
|----------------|--------------|------------------|
| `detectRootManagementApps` | Détecte les apps de gestion root (Magisk, SuperSU) | `false` |
| `detectPotentiallyDangerousApps` | Détecte des apps potentiellement dangereuses | `false` |
| `detectTestKeys` | Vérifie si le build est signé avec des test-keys | `false` |
| `checkForBusyBoxBinary` | Cherche le binaire BusyBox dans le système | `false` |
| `checkForSuBinary` | Cherche le binaire `su` dans les chemins système | `false` |
| `checkSuExists` | Tente d'exécuter `su` via Runtime.exec | `false` |
| `checkForRWPaths` | Vérifie si des partitions système sont montées en écriture | `false` |
| `checkForDangerousProps` | Vérifie les propriétés système dangereuses (`ro.debuggable`, etc.) | `false` |
| `checkForRootNative` | Effectue un check root au niveau natif (JNI) | `false` |
| `detectRootCloakingApps` | Détecte les apps de camouflage de root | `false` |
| `Utils.isSelinuxFlagInEnabled` | Vérifie si SELinux est en mode permissif | `false` |
| `checkForMagiskBinary` | Cherche le binaire Magisk | `false` |
| `isRooted` | Méthode principale qui agrège tous les checks | `false` |

Le mécanisme utilisé est `overload().implementation` : Frida remplace le corps de chaque méthode Java par une fonction JavaScript personnalisée qui retourne directement `false` sans exécuter la logique originale.

---

## Avant le bypass — Root détecté

Sans injection Frida, Genymotion étant rooté par défaut, RootBeer Sample détecte le root et affiche **ROOTED** en rouge. Plusieurs checks échouent :

- ❌ BusyBoxBinary
- ❌ SU Binary
- ❌ 2nd SU Binary check
- ❌ Dangerous Props
- ❌ Root via native check
<img width="448" height="934" alt="3" src="https://github.com/user-attachments/assets/3ca93aad-44e5-4f1a-bc86-75f93adecef0" />


---

## Lancement du bypass avec Frida

Frida client est installé sur Windows (version 17.9.1) et Frida server tourne déjà sur l'émulateur Genymotion. La commande suivante injecte le script dans l'application au moment de son démarrage :

```powershell
frida -U -f com.scottyab.rootbeer.sample -l C:\Users\PC\frida-scripts\bypass_rootbeer.js
```

<img width="1069" height="343" alt="2" src="https://github.com/user-attachments/assets/614e970f-bffd-45e3-9489-a4c8bb71ae63" />

---

## Après le bypass — Root non détecté

Après injection du script, toutes les méthodes de détection retournent `false`. RootBeer Sample affiche **NOT ROOTED** en vert avec tous les checks passés :

- ✅ Root Management Apps
- ✅ Potentially Dangerous Apps
- ✅ Root Cloaking Apps
- ✅ TestKeys
- ✅ BusyBoxBinary
- ✅ SU Binary
- ✅ 2nd SU Binary check
- ✅ For RW Paths
- ✅ Dangerous Props
- ✅ Root via native check
- ✅ SE Linux Flag Is Enabled
- ✅ Magisk specific checks

<img width="462" height="934" alt="1" src="https://github.com/user-attachments/assets/0c0c6583-5a6a-4804-b451-217fe7379864" />

---

## Conclusion

Ce lab démontre qu'une détection de root purement basée sur des vérifications Java, aussi complète soit-elle, peut être entièrement neutralisée par Frida en quelques lignes de JavaScript. Chaque méthode de la bibliothèque RootBeer a été hookée individuellement, ce qui illustre l'importance de combiner les vérifications Java avec des mécanismes natifs et des protections anti-tampering plus robustes (certificate pinning, integrity checks, obfuscation).

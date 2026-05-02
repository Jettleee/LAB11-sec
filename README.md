# 🎯 Rapport d'Exploitation : Contournement des Mécanismes Anti-Root (Uncrackable Level 2)

**Analyste en Cyber-Défense :** Youssef CHARAF
**Cible Audité :** `owasp.mstg.uncrackable2` (OWASP)
**Vecteur d'attaque :** Instrumentation Dynamique via Frida

---

## 🔎 1. Contexte et Stratégie d'Attaque

L'application Uncrackable Level 2 de l'OWASP intègre des défenses agressives contre les environnements compromis. L'objectif de cette mission est d'étudier ces mécanismes de détection de root et d'élaborer une chaîne d'exploitation dynamique (hooks) pour les neutraliser à la volée. L'idée est de rendre notre environnement "invisible" aux yeux des sondes de l'application.

---

## 🛠️ 2. Mise en Place de l'Infrastructure d'Interception

Avant toute tentative de manipulation, l'infrastructure de communication entre le client (PC) et le démon (`frida-server` sur l'émulateur) a été établie et vérifiée. Le binaire serveur a été poussé via ADB et exécuté avec les privilèges requis en arrière-plan.

**Validation de la télémétrie :**
<img width="732" height="611" alt="image" src="https://github.com/user-attachments/assets/7efc0c98-23ca-4948-a0ba-b48e8d73f7bf" />


---

## 🛑 3. Analyse du Comportement Nominal (La Défense)

Pour comprendre la logique de l'application, une exécution standard (sans instrumentation) a été lancée. 

**Résultat :** Le mécanisme d'autoprotection se déclenche immédiatement. Une alerte "Root detected!" apparaît, bloquant l'accès à l'interface principale. La validation de cette popup entraîne un crash volontaire du processus.

<img width="471" height="955" alt="image" src="https://github.com/user-attachments/assets/90332d17-eaaa-4f21-8b1b-94692c5ab815" />


---

## ☕ 4. Tentative de Bypass : Couche Haute (API Java)

La première hypothèse d'attaque consistait à neutraliser les contrôles de haut niveau. Un script d'injection (`bypass_root.js`) a été utilisé pour falsifier les propriétés système (`Build.TAGS`), intercepter les requêtes de fichiers (`File.exists`) et bloquer l'exécution de sous-processus via `Runtime.exec`.


**Diagnostic :** Échec. Malgré la bonne exécution des hooks sur la machine virtuelle Java, l'application s'est tout de même fermée. Conclusion immédiate : la détection critique s'opère dans les couches basses du système (JNI/Code Natif).

---

## 🔬 5. Reverse Engineering Dynamique (Traçage Natif)

Face à cette défense en profondeur, une analyse dynamique des appels systèmes a été initiée via `frida-trace`. L'objectif était de "sniffer" les requêtes natives liées au système de fichiers.



**Découverte :** La trace a révélé que la bibliothèque native de l'application interrogeait directement le système d'exploitation via les fonctions `access` et `open` pour vérifier la présence des binaires de compromission (notamment `/system/bin/su` et `/system/xbin/su`).

<img width="1212" height="151" alt="image" src="https://github.com/user-attachments/assets/50d76c88-baba-4854-83bb-b5a019f8a5df" />

<img width="1140" height="515" alt="image" src="https://github.com/user-attachments/assets/50a93d53-f875-4c4a-a791-579d798cf5dc" />



---

## 💥 6. Exploitation Réussie : Interception JNI

Forts de cette intelligence technique, nous avons déployé un second script (`bypass_native.js`). Ce script utilise la classe `Interceptor` de Frida pour se greffer directement sur les fonctions exportées par la `libc.so`. 

Lorsqu'un appel à `open` ou `access` cible un chemin surveillé, notre hook intercepte la requête et renvoie artificiellement un code d'erreur (pointeur `-1`), simulant l'absence totale du fichier sur le disque.

**Déploiement de la charge combinée :**


<img width="1903" height="428" alt="image" src="https://github.com/user-attachments/assets/b8e36a9d-1e50-4f58-a57f-be1cd3ce1ce5" />

<img width="474" height="964" alt="image" src="https://github.com/user-attachments/assets/a8c3d9f7-1d1e-4453-981b-8942243ae2ea" />


---

## 🏁 7. Conclusion de l'Audit

Ce laboratoire met en lumière la nécessité d'une approche d'analyse hybride. Les développeurs de l'OWASP ont intelligemment masqué leur logique de détection de root dans le code compilé (C/C++) pour éviter les contournements Java trop triviaux. 

La maîtrise conjointe du hooking sur l'API Android (Java) et de l'interception au niveau de la librairie standard (C) a été indispensable pour aveugler complètement les mécanismes de sécurité de l'application cible et obtenir un environnement de test persistant.

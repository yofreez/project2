# Mise en place d'un mini-SOC Wazuh : détection de Lumma et RedLine avec Sigma


## 1. Disclamer 

Petit rappel : suite à notre entretien de jeudi dernier, j’ai changé de projet. Auparavant, je n’avais pas ajouté mon rapport sur Git, ce qui explique pourquoi je dépose tout en un seul bloc.

À partir de maintenant, je reprends le suivi comme demandé, avec des ajouts et des commits réguliers.

Concernant l’utilisation de l’intelligence artificielle, étant donné qu’aucun commit n’avait été effectué pour le rapport auparavant, les différentes utilisations de l’IA sont retracées dans le journal IA, notamment dans les sections IA 1 et IA 2.

## 2. Étude préalable des malwares et création des règles Sigma

Avant le déploiement , une étude comparative de **six échantillons** a été réalisée : trois Lumma et trois RedLine.

Les analyses se sont appuyées sur des rapports de sandbox (ANY.RUN et VirusTotal) et des captures réseau PCAP examinées avec Wireshark et tshark.

Les malwares n'ont pas été exécutés sur les machines de travail.

### 2.1. Comportements retenus

**Lumma Stealer**

- Lancement observé de `chrome.exe` avec une URL contenant `/api/set_agent`.
- Présence des paramètres `token=` et `agent=Chrome`.
- Communications HTTP et transferts binaires.

**RedLine Stealer**

- Requêtes HTTP SOAP/XML.
- Présence des en-têtes `SOAPAction`.
- Utilisation de `tempuri.org/Endpoint/`.
- Opérations `CheckConnect`, `EnvironmentSettings`, `SetEnvironment` et `GetUpdates`.

### 2.2. Validation initiale des règles Sigma

Les règles Sigma ont d'abord été validées hors ligne avec `sigma-cli` et Chainsaw.

**Machine : Fedora Terminal Linux**

Exemple de conversion de la règle Lumma :

```bash
sigma convert -t splunk -p splunk_windows ~/projet-stealer/sigma/lumma_set_agent.yml
```

La règle Sigma Lumma a été validée sur un événement Sysmon contrôlé, avec deux tests négatifs.

La règle RedLine a détecté **8 requêtes SOAP réelles** extraites du PCAP REDLINE-01, sans détection sur les événements bénins du test négatif.

---

## 3. Mise en place de l'infrastructure virtuelle

Le laboratoire a été déployé sur un hôte **Fedora Linux** utilisant **KVM/libvirt**.

### 3.1. Architecture

| Machine | Rôle | Adresse |
|---|---|---|
| Fedora | Hôte de virtualisation et administration | Passerelle libvirt `192.168.122.1` |
| VM Wazuh | Manager, analyse des alertes et dashboard | `192.168.122.207` |
| VM Windows Server 2022 Core | Machine supervisée | `192.168.122.224` |

Les machines virtuelles utilisent le réseau libvirt `default` en NAT.

### 3.2. Vérification des machines virtuelles

**Machine : Fedora Terminal Linux**

```bash
sudo virsh list --all
```

Affichage des adresses attribuées par DHCP :

```bash
sudo virsh net-dhcp-leases default
```

### 3.3. Connexion au serveur Wazuh

**Machine : Fedora Terminal Linux**

```bash
ssh wazuh-user@192.168.122.207
```

Passage en administrateur :

```bash
su -
```

### 3.4. Vérification du Manager Wazuh

**Machine : VM Wazuh Terminal root**

```bash
rpm -q wazuh-manager
```

Résultat : version **4.14.8**.

Vérification du service :

```bash
systemctl is-active wazuh-manager
```

Résultat :

```text
active
```

Le dashboard est accessible à l'adresse :

`https://192.168.122.207/`

---

## 4. Installation et connexion de l'agent Wazuh sur Windows

Sur Windows Server Core, l'agent Wazuh 4.14.8 a été installé en PowerShell.

### 4.1. Installation de l'agent

**Machine : VM Windows Server PowerShell administrateur**

IA 1:

Après téléchargement du MSI dans `C:\Temp`, :

```powershell
Start-Process msiexec.exe -ArgumentList '/i C:\Temp\wazuh-agent.msi /qn WAZUH_MANAGER=192.168.122.207 WAZUH_REGISTRATION_SERVER=192.168.122.207 WAZUH_AGENT_NAME=WIN-SOC-LAB /l*v C:\Temp\wazuh-install.log' -Wait -PassThru
```

Cette commande configure :

- Le Manager Wazuh : `192.168.122.207`.
- Le serveur d'enregistrement : `192.168.122.207`.
- Le nom de l'agent : `WIN-SOC-LAB`.

### 4.2. Vérification du service

```powershell
Get-Service WazuhSvc
```

Résultat :

```text
Running
```

**Validation :** l'agent `WIN-SOC-LAB` apparaît avec le statut **Active** dans le dashboard Wazuh.

---

## 5. Installation de Sysmon et collecte des événements

Sysmon a été installé sur Windows Server Core pour enregistrer les activités système.

### 5.1. Installation de Sysmon

**Machine : VM Windows Server PowerShell administrateur**

Extraction de l'archive téléchargée :

```powershell
Expand-Archive -Path "C:\Tools\Sysmon\Sysmon.zip" -DestinationPath "C:\Tools\Sysmon" -Force
```

Installation :

```powershell
Set-Location C:\Tools\Sysmon
.\Sysmon64.exe -accepteula -i
```

Vérification :

```powershell
Get-Service Sysmon64
```

Résultat : `Running`.

### 5.2. Premier test de création de processus

Un test a été réalisé avec `whoami.exe`.

```powershell
Start-Process "$env:WINDIR\System32\whoami.exe" -ArgumentList "/all" -Wait
```

Consultation des événements Sysmon ID 1 :

```powershell
Get-WinEvent -FilterHashtable @{
    LogName = 'Microsoft-Windows-Sysmon/Operational'
    Id = 1
} -MaxEvents 5 | Format-List TimeCreated, Id, Message
```

**Résultat :** Sysmon enregistre correctement :

- Le processus exécuté.
- Sa ligne de commande.
- Le processus parent.
- L'utilisateur.
- Le hash SHA-256.

### 5.3. Intégration de Sysmon dans Wazuh

Le fichier de configuration de l'agent Windows est :

```text
C:\Program Files (x86)\ossec-agent\ossec.conf
```

Le bloc suivant a été ajouté :

```xml
<localfile>
  <location>Microsoft-Windows-Sysmon/Operational</location>
  <log_format>eventchannel</log_format>
</localfile>
```

Après redémarrage du service Wazuh :

```powershell
Restart-Service WazuhSvc
```

Le journal de l'agent confirmait :

```text
Analyzing event log: 'Microsoft-Windows-Sysmon/Operational'.
```

### 5.4. Enrichissement de la télémétrie

La configuration Sysmon a ensuite été enrichie pour surveiller davantage d'activités.

Les événements suivants ont été observés :

| Event ID | Information |
|---|---|
| 1 | Création de processus |
| 3 | Connexion réseau |
| 5 | Fin de processus |
| 11 | Création de fichiers |
| 12 | Création ou suppression de clés/valeurs du registre |
| 22 | Requêtes DNS |

Les événements réseau et DNS ont été validés avec :

```powershell
Test-NetConnection -ComputerName 192.168.122.207 -Port 1514
```

```powershell
Resolve-DnsName example.com
```

**Résultat :** Sysmon a enregistré la connexion TCP vers Wazuh et la requête DNS vers `example.com`.

---

## 6. Première validation d'une règle personnalisée Wazuh

Avant d'intégrer les règles Sigma, une règle simple a été créée pour vérifier le fonctionnement du moteur de détection Wazuh.

### 6.1. Création de la règle de test

**Machine : VM Wazuh Terminal root**

Fichier :

```text
/var/ossec/etc/rules/sysmon_lab_rules.xml
```

Règle principale :

```xml
<rule id="100151" level="5">
  <if_sid>61603</if_sid>
  <field name="win.eventdata.image" type="pcre2">(?i)whoami\.exe$</field>
  <description>LAB - Sysmon whoami.exe detected</description>
</rule>
```

Cette règle s'appuie sur la règle native Sysmon `61603` et détecte l'exécution de `whoami.exe`.

### 6.2. Validation et activation

```bash
/var/ossec/bin/wazuh-analysisd -t
```

Redémarrage :

```bash
systemctl restart wazuh-manager
```

**Résultat :** la règle `100151` a généré une alerte visible dans le dashboard Wazuh.

La chaîne complète a ainsi été validée :

**Windows Server → Sysmon → Agent Wazuh → Manager → Règle personnalisée → Dashboard**

---

## 7. Intégration et validation de la détection Sigma Lumma

La règle Sigma `lumma_set_agent.yml`, créée lors de l'étude préalable, a été adaptée au format Wazuh XML.

### 7.1. Création de la règle Lumma

**Machine : VM Wazuh — Terminal root**

Fichier :

```text
/var/ossec/etc/rules/lumma_rules.xml
```

Règle :

```xml
<rule id="100152" level="10">
  <if_sid>61603</if_sid>
  <field name="win.eventdata.image" type="pcre2">(?i)\\chrome\.exe$</field>
  <field name="win.eventdata.commandLine" type="pcre2">(?i)(?=.*\/api\/set_agent)(?=.*token=)(?=.*agent=Chrome).*</field>
  <description>LAB - Lumma Sigma - Chrome set_agent suspicious command line</description>
</rule>
```

La règle impose deux conditions :

1. Le processus doit se terminer par `chrome.exe`.
2. La ligne de commande doit contenir `/api/set_agent`, `token=` et `agent=Chrome`.

La configuration a été validée avec `wazuh-analysisd -t`, puis le Manager redémarré.

### 7.2. Test positif Lumma

**Machine : VM Windows Server PowerShell administrateur**

Pour tester la détection sans exécuter de malware, une copie inoffensive de `whoami.exe` a été renommée `chrome.exe`.

```powershell
Copy-Item "$env:WINDIR\System32\whoami.exe" "C:\Tools\SigmaLab\chrome.exe"
```

Lancement avec les paramètres recherchés :

```powershell
Start-Process -FilePath "C:\Tools\SigmaLab\chrome.exe" `
  -ArgumentList 'https://example.invalid/api/set_agent?token=LABTEST&agent=Chrome' -Wait
```

Cette commande ne contacte aucun serveur malveillant. Elle sert uniquement à générer un événement Sysmon contenant les paramètres attendus.

### 7.3. Résultat dans Wazuh

Une alerte est apparue dans **Threat Hunting → Events**.

Informations confirmées :

| Champ | Valeur |
|---|---|
| Agent | `WIN-SOC-LAB` |
| Règle | `100152` |
| Niveau | 10 |
| Source | Sysmon Event ID 1 |
| Processus | `C:\Tools\SigmaLab\chrome.exe` |
| Ligne de commande | Contient les trois indicateurs |

Le champ `originalFileName` indiquait `whoami.exe`, confirmant l'utilisation du programme bénin renommé.

### 7.4. Tests négatifs Lumma

Deux tests négatifs ont ensuite été effectués :

**Test 1 :** lancement de `chrome.exe` avec une URL normale.

**Test 2 :** lancement de `chrome.exe` avec `/api/set_agent`, mais sans les autres paramètres nécessaires.

Le compteur des alertes a été vérifié sur Wazuh :

```bash
grep -c '"id":"100152"' /var/ossec/logs/alerts/alerts.json
```

Résultat avant les tests :

```text
2
```

Résultat après les tests :

```text
2
```

**Conclusion :** aucune nouvelle alerte n'a été déclenchée lors des deux tests négatifs.

La règle Lumma a donc été validée sur un scénario positif et deux scénarios négatifs.

Cette validation repose sur une simulation bénigne et non sur l'exécution d'un véritable Lumma.

---

## 8. Intégration et validation de la détection Sigma RedLine

La détection RedLine repose sur les communications HTTP SOAP/XML observées dans les captures réseau de l'étude précédente.

Contrairement à la détection Lumma, **Sysmon ID 3 ne contient pas les en-têtes HTTP** nécessaires à cette règle.

Les événements ont donc été extraits d'un véritable PCAP RedLine et préparés au format JSON.

### 8.1. Préparation des événements RedLine

**Machine : Fedora — Terminal Linux**

Le fichier source est :

```text
~/projet-stealer/sigma/redline-real.json
```

Il contient huit événements HTTP issus de REDLINE-01.

Ce fichier a été converti au format **JSON Lines**, avec un événement JSON par ligne.

Fichier obtenu :

```text
~/projet-stealer/sigma/redline-wazuh.jsonl
```

### 8.2. Transfert vers Wazuh

**Machine : Fedora Terminal Linux**

```bash
scp ~/projet-stealer/sigma/redline-wazuh.jsonl \
  wazuh-user@192.168.122.207:/home/wazuh-user/
```

**Machine : VM Wazuh — Terminal root**

Création d'un dossier dédié :

```bash
mkdir -p /var/ossec/logs/lab
```

Copie du fichier :

```bash
cp /home/wazuh-user/redline-wazuh.jsonl /var/ossec/logs/lab/
```

Vérification :

```bash
wc -l /var/ossec/logs/lab/redline-wazuh.jsonl
```

Résultat :

```text
8 /var/ossec/logs/lab/redline-wazuh.jsonl
```

### 8.3. Création de la règle RedLine

**Machine : VM Wazuh Terminal root**

Fichier :

```text
/var/ossec/etc/rules/redline_rules.xml
```

Règle :

```xml
<rule id="100153" level="10">
  <decoded_as>json</decoded_as>
  <field name="provider">^RedLine-HTTP$</field>
  <field name="raw" type="pcre2">(?i)SOAPAction.*tempuri\.org/Endpoint/(CheckConnect|EnvironmentSettings|SetEnvironment|GetUpdates)</field>
  <description>LAB - RedLine Sigma - Suspicious SOAP communication</description>
</rule>
```

Cette règle recherche :

- La présence de `SOAPAction`.
- La structure `tempuri.org/Endpoint/`.
- Une opération parmi `CheckConnect`, `EnvironmentSettings`, `SetEnvironment` et `GetUpdates`.

Elle ne dépend pas d'une adresse IP spécifique.

### 8.4. Premier test avec wazuh-logtest

**Machine : VM Wazuh  Terminal root**

```bash
head -n 1 /var/ossec/logs/lab/redline-wazuh.jsonl | /var/ossec/bin/wazuh-logtest
```

Résultat :

```text
Phase 2: Completed decoding.
name: 'json'

Phase 3: Completed filtering (rules).
id: '100153'
level: '10'
description: 'LAB - RedLine Sigma - Suspicious SOAP communication'

**Alert to be generated.
```

**Conclusion :** le moteur de test Wazuh a correctement reconnu le premier événement RedLine.

### 8.5. Configuration de la collecte RedLine

Pour rendre les alertes visibles dans le dashboard, un fichier de surveillance a été préparé :

```text
/var/ossec/logs/lab/redline-stream.jsonl
```

Le fichier `/var/ossec/etc/ossec.conf` a été sauvegardé avant modification.

Un bloc de configuration a ensuite été ajouté :

```xml
<ossec_config>
  <localfile>
    <log_format>json</log_format>
    <location>/var/ossec/logs/lab/redline-stream.jsonl</location>
  </localfile>
</ossec_config>
```

Après redémarrage du Manager, Wazuh était prêt à collecter les événements JSON ajoutés à ce fichier.

### 8.6. Test positif RedLine

**Machine : VM Wazuh Terminal root**

Un premier événement a été envoyé :

```bash
head -n 1 /var/ossec/logs/lab/redline-wazuh.jsonl >> /var/ossec/logs/lab/redline-stream.jsonl
```

Une alerte `100153` est apparue dans le dashboard.

Les huit événements ont ensuite été rejoués :

```bash
cat /var/ossec/logs/lab/redline-wazuh.jsonl >> /var/ossec/logs/lab/redline-stream.jsonl
```

Le compteur d'alertes a été vérifié avec :

```bash
grep -c '"id":"100153"' /var/ossec/logs/alerts/alerts.json
```

**Résultat : le compteur a augmenté de 8 alertes.**

Les événements étaient visibles dans **Threat Hunting → Events**.

### 8.7. Test négatif RedLine

AI 2

Un événement SOAP bénin contenant une opération fictive `HealthCheck` a été envoyé.

```bash
printf '%s\n' '{"provider":"RedLine-HTTP","raw":"SOAPAction: \"http://tempuri.org/Endpoint/HealthCheck\"","lab_source":"REDLINE-NEGATIVE-TEST"}' >> /var/ossec/logs/lab/redline-stream.jsonl
```

Cet événement contient la structure SOAP, mais aucune des quatre opérations RedLine recherchées.

Le compteur d'alertes `100153` est resté identique.

**Conclusion :** le test négatif est réussi.

La détection RedLine a donc été validée sur huit événements réels issus d'une capture réseau et sur un événement négatif.

Il s'agit d'une analyse de traces historiques rejouées, et non d'une détection en direct d'un RedLine actif.

---

## 9. Bilan des résultats

| Élément | Résultat |
|---|---|
| Wazuh Manager et agent Windows | Opérationnels |
| Collecte Sysmon | Validée |
| Règle Wazuh de test `100151` | Validée |
| Règle Lumma `100152` | Test positif et deux tests négatifs validés |
| Règle RedLine `100153` | 8 alertes sur PCAP réel, test négatif validé |
| Remédiation et confinement | Non réalisés |
| Exécution de vrais malwares dans le nouveau laboratoire | Non réalisée |

### 9.1. Résultats de la détection Lumma

La règle Sigma Lumma a été adaptée en règle Wazuh `100152`.

Elle détecte la combinaison de paramètres recherchée dans les événements de création de processus Sysmon.

Les tests réalisés montrent :

- Une détection positive réussie.
- Deux tests négatifs sans nouvelle alerte.
- Une alerte de niveau 10 dans le dashboard.

### 9.2. Résultats de la détection RedLine

La règle Sigma RedLine a été adaptée en règle Wazuh `100153`.

Elle détecte les opérations SOAP/XML présentes dans les événements issus du PCAP REDLINE-01.

Les tests réalisés montrent :

- Huit nouvelles alertes après l'importation des huit événements.
- Un test négatif sans nouvelle alerte.
- Des alertes de niveau 10 visibles dans le dashboard.

---


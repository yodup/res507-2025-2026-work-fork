# Lab 52 · Architecture notes

> Système : `quote-app` (Node.js, port 3000) sur k3s, namespace `quote-lab`, dans une VM Ubuntu.

---

## 1. Schéma A : « tourne dans » (frontières)

![Schéma A : tourne dans (frontières)](architecture-a.png)

<details>
<summary>Version Mermaid (s'affiche sur GitHub)</summary>

```mermaid
flowchart TB
  subgraph HW["MATERIEL DE L'HOTE : PC physique (CPU, RAM, disque)"]
    subgraph HV["HYPERVISEUR : VirtualBox"]
      subgraph VM["VM UBUNTU = noeud Kubernetes (k3s) + noyau Linux invite"]
        subgraph NS["Namespace quote-lab"]
          subgraph APPD["Deployment quote-app : 3 replicas"]
            P1["Pod 1<br/>conteneur quote-app"]
            P2["Pod 2<br/>conteneur quote-app"]
            P3["Pod 3<br/>conteneur quote-app"]
          end
          subgraph DBD["Deployment quote-db : PLANIFIE / a confirmer"]
            DBP["Pod PostgreSQL<br/>conteneur postgres"]
          end
          PVC[("PVC quote-db-data<br/>local-path : disque de la VM")]
          DBP --- PVC
        end
      end
    end
  end

  classDef pod fill:#bbf7d0,stroke:#15803d,stroke-width:2px,color:#052e16
  classDef db fill:#fed7aa,stroke:#c2410c,stroke-width:2px,stroke-dasharray:5 3,color:#431407
  classDef storage fill:#fef08a,stroke:#a16207,stroke-width:2px,color:#422006

  class P1,P2,P3 pod
  class DBP db
  class PVC storage

  style HW fill:#f1f5f9,stroke:#64748b,stroke-width:2px,color:#0f172a
  style HV fill:#dbeafe,stroke:#2563eb,stroke-width:2px,color:#1e3a8a
  style VM fill:#ede9fe,stroke:#7c3aed,stroke-width:3px,color:#3b0764
  style NS fill:#faf5ff,stroke:#a78bfa,stroke-width:1px,color:#3b0764
  style APPD fill:#f0fdf4,stroke:#16a34a,stroke-width:2px,color:#14532d
  style DBD fill:#fff7ed,stroke:#ea580c,stroke-width:2px,stroke-dasharray:5 3,color:#7c2d12
```

</details>

---

## 2. Schéma B : chemins de trafic

![Schéma B : chemins de trafic](architecture-b.png)

<details>
<summary>Version Mermaid (s'affiche sur GitHub)</summary>

```mermaid
flowchart LR
  BR["Navigateur du PC<br/>localhost:8080"]
  TUN["Acces PC vers VM<br/>VS Code Ports / tunnel SSH"]
  PF["kubectl port-forward<br/>TEMPORAIRE : 8080 vers 3000"]
  SC["Pod service-check<br/>client in-cluster (Lab 42)"]
  SVCA["Service quote-app<br/>port 80 vers 3000"]
  SVCD["Service quote-db<br/>port 5432"]

  subgraph PODS["Pods quote-app"]
    A1["Pod A"]
    A2["Pod B"]
    A3["Pod C"]
  end

  DB["Pod PostgreSQL<br/>PLANIFIE / a confirmer"]
  VOL[("PVC")]

  BR --> TUN
  TUN --> PF
  PF -. "1 seul Pod cible : PAS de load balancing" .-> A1

  SC -->|"http://quote-app/health<br/>DNS + Service"| SVCA
  SVCA --> A1
  SVCA --> A2
  SVCA --> A3

  A1 --> SVCD
  A2 --> SVCD
  A3 --> SVCD
  SVCD --> DB
  DB --- VOL

  classDef user fill:#e2e8f0,stroke:#475569,stroke-width:2px,color:#0f172a
  classDef temp fill:#fecaca,stroke:#b91c1c,stroke-width:2px,stroke-dasharray:5 3,color:#450a0a
  classDef client fill:#99f6e4,stroke:#0f766e,stroke-width:2px,color:#042f2e
  classDef svc fill:#bfdbfe,stroke:#1d4ed8,stroke-width:2px,color:#172554
  classDef pod fill:#bbf7d0,stroke:#15803d,stroke-width:2px,color:#052e16
  classDef db fill:#fed7aa,stroke:#c2410c,stroke-width:2px,stroke-dasharray:5 3,color:#431407
  classDef storage fill:#fef08a,stroke:#a16207,stroke-width:2px,color:#422006

  class BR,TUN user
  class PF temp
  class SC client
  class SVCA,SVCD svc
  class A1,A2,A3 pod
  class DB db
  class VOL storage

  style PODS fill:#f0fdf4,stroke:#16a34a,stroke-width:2px,color:#14532d

  linkStyle 0,1 stroke:#ea580c,stroke-width:3px
  linkStyle 2 stroke:#dc2626,stroke-width:3px
  linkStyle 3,4,5,6 stroke:#0d9488,stroke-width:4px
  linkStyle 7,8,9,10,11 stroke:#7c3aed,stroke-width:3px
```

</details>

**Lecture, trois trajets distincts :**

1. **Utilisateur** : navigateur, tunnel, port-forward, puis **un seul Pod**. Ce chemin ne démontre ni la répartition de charge ni la sélection des Pods Ready.
2. **In-cluster (turquoise épais)** : `service-check`, Service, puis les Pods Ready. Il passe par le DNS du cluster : c'est celui que vous avez vérifié au Lab 42.
3. **Base de données (violet)** : les Pods app, Service `quote-db`, PostgreSQL, PVC.

PostgreSQL est marqué **planifié / à confirmer** : retirez cette mention une fois le déploiement vérifié. Toute démonstration faite par l'instructeur doit être étiquetée **observed**.

> Les schémas sont des images PNG (`architecture-a.png`, `architecture-b.png`) : gardez-les dans le même dossier que ce fichier pour qu'ils s'affichent dans l'aperçu VS Code et sur GitHub. Vous pouvez aussi recopier un schéma à la main et ajouter une photo.

---

## 3. Notes de frontière

**Noyau partagé.** Les conteneurs de l'app et de la base partagent le noyau Linux de la VM Ubuntu ; l'hyperviseur fournit la frontière de la VM (isolation vis-à-vis de l'hôte), et Kubernetes gère les workloads à l'intérieur de cette VM.

**Stockage et pannes.** Un Pod de l'app peut être remplacé car il est sans état, et la citation survit au remplacement du Pod PostgreSQL car ses données sont sur le PVC ; en revanche, trois réplicas avec un stockage `local-path` sur une seule VM ne protègent pas contre la perte de la VM ou de son disque. Sauvegardes et conception multi-nœuds restent du travail futur, non fournis par ce cluster d'enseignement.

---

## 4. Ce qui survit à quoi

| Événement | Survit ? | Pourquoi |
|---|---|---|
| Suppression d'un Pod app | Oui | Le ReplicaSet recrée un Pod (nouveau nom), l'app est sans état |
| Suppression du Pod PostgreSQL | Oui (données) | Les données sont sur le PVC, remonté dans le nouveau Pod |
| Perte de la VM ou de son disque | **Non** | Un seul nœud, stockage `local-path` local, pas de sauvegarde |

---

## 5. Preuves

| Contrôle | Résultat |
|---|---|
| PVC `quote-db-data` | `Bound` : pvc-7f53ab9f-a31a-44bb-bc4a-c16a851f4cf2 |
| Pod PostgreSQL | `quote-db-...` : 7cb5c66d76-4r8wz |
| Citation (marqueur) | <div>res509-1792400679</div> |

# CityPilot — Gestion des services techniques communaux

Application web mono-fichier (`index.html`), hébergeable sur GitHub Pages, pour le monitoring et le reporting du patrimoine communal.

## Modules inclus

| Module | Contenu |
|---|---|
| Tableau de bord | KPI, alertes centralisées, derniers signalements |
| Arbres | Cartographie, essence, état sanitaire, année de plantation |
| Mobilier urbain | Cartographie, type, état, année de pose |
| Voirie & interventions | Carte des interventions, priorité, statut, **historique horodaté** |
| Éclairage public | Armoires de commande, points lumineux, horloge, extinction nocturne |
| Bornes incendie | Réf. PI/BI, diamètre, pression, date de contrôle (alerte si > 1 an) |
| Défibrillateurs | **Alerte à 1 mois** de la péremption électrodes et batterie |
| Flotte véhicules | Fiche véhicule, travaux effectués, **alertes CT / révision / assurance à 30 j** |
| Bâtiments | Surface, type et catégorie ERP, énergie, commission de sécurité |
| Signalements citoyens | Page publique sans connexion → crée une alerte, convertible en intervention voirie |
| Administration | Gestion des communes clientes, statut d'abonnement |

Chaque module cartographique permet :
- la **saisie point par point** (clic sur la carte) ;
- l'**import GeoJSON depuis QGIS** (couche de points, projection WGS 84 / EPSG:4326 — dans QGIS : clic droit sur la couche → Exporter → Sauvegarder les entités sous → GeoJSON → SCR `EPSG:4326`) ;
- l'export CSV du tableau.

## Comptes de démonstration

- Client : `VALMONT37` / `demo`
- Admin : `ADMIN` / `admin`

## Déploiement GitHub Pages

1. Créer un dépôt (ex. `citypilot`), y pousser `index.html`.
2. Settings → Pages → Source : branche `main`, dossier `/ (root)`.
3. L'app est en ligne sur `https://<utilisateur>.github.io/citypilot/`.

## Passage en production

⚠️ En mode démo, les données vivent en mémoire (bouton d'export/restauration JSON dans Paramètres). GitHub Pages étant un hébergement **statique**, la persistance, l'authentification réelle et le paiement passent par Firebase + Stripe.

### 1. Firebase (Auth + Firestore)

```html
<script type="module">
import { initializeApp } from "https://www.gstatic.com/firebasejs/10.12.0/firebase-app.js";
import { getAuth, signInWithEmailAndPassword } from "https://www.gstatic.com/firebasejs/10.12.0/firebase-auth.js";
import { getFirestore, doc, getDoc, setDoc } from "https://www.gstatic.com/firebasejs/10.12.0/firebase-firestore.js";

const app = initializeApp(FIREBASE_CONFIG);
const auth = getAuth(app);
const db = getFirestore(app);

// Connexion : le "code client" devient un email technique, ex. valmont37@citypilot.app
// Chargement : const snap = await getDoc(doc(db, "clients", codeClient));
// Sauvegarde : await setDoc(doc(db, "clients", codeClient), DB);
</script>
```

Règles Firestore (isolation par client) :

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /clients/{clientId}/{document=**} {
      allow read, write: if request.auth != null
        && request.auth.token.clientId == clientId
        && get(/databases/$(database)/documents/clients/$(clientId)).data.abo in ['actif','essai'];
    }
    // Signalements publics : écriture seule, sans authentification
    match /clients/{clientId}/signalements/{sigId} {
      allow create: if true;
    }
  }
}
```

⚠️ Les mots de passe ne doivent **jamais** être stockés en clair côté client comme dans la démo — c'est Firebase Auth qui les gère.

### 2. Stripe (abonnements)

GitHub Pages ne peut pas exécuter de code serveur : le paiement s'appuie sur Stripe + une Cloud Function Firebase.

1. Créer les produits/tarifs dans Stripe (abonnement annuel par strate de population).
2. Générer un **Payment Link** par formule et le renseigner dans `STRIPE_PAYMENT_LINK` (ou utiliser Stripe Checkout).
3. Déployer une Cloud Function qui reçoit les webhooks :
   - `invoice.paid` → `abo = "actif"`, mise à jour de l'échéance ;
   - `invoice.payment_failed` / `customer.subscription.deleted` → `abo = "suspendu"`.
4. À la connexion, l'app refuse les comptes `suspendu` (déjà implémenté).

```js
// functions/index.js (extrait)
exports.stripeWebhook = onRequest(async (req, res) => {
  const event = stripe.webhooks.constructEvent(req.rawBody, req.headers["stripe-signature"], WEBHOOK_SECRET);
  const clientId = event.data.object.metadata.clientId;
  if (event.type === "invoice.paid")
    await db.doc(`clients/${clientId}`).update({ abo: "actif", echeance: nextYear() });
  if (event.type === "customer.subscription.deleted")
    await db.doc(`clients/${clientId}`).update({ abo: "suspendu" });
  res.sendStatus(200);
});
```

## Pistes d'évolution

- Photos jointes aux signalements et aux fiches (Firebase Storage)
- Rappels par email des échéances (Cloud Functions + Trigger Email)
- Couches linéaires (voiries, réseaux) en plus des points
- Fond de plan cadastral / orthophoto IGN (Géoplateforme WMTS)
- Rapports PDF annuels par module (bilan patrimoine pour le conseil municipal)
- Rôles multiples par commune (DST, agents, élus en lecture seule)

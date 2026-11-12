# Ciné-Résa — Jalon V0

## 1. Présentation
Ciné-Résa est un système de réservation de places de cinéma conçu en microservices Spring Boot. Le spectateur consulte les films et séances, réserve des places, paie et reçoit un billet avec QR code. Une annulation de séance entraîne l'annulation des réservations et les remboursements nécessaires.

**Unité métier :** Réservation. États principaux : `EN_ATTENTE_PAIEMENT`, `CONFIRMEE`, `ANNULEE`, `REMBOURSEE`.

## 2. Objectifs de V0
- Définir les responsabilités des services et leur autonomie de données.
- Décrire les échanges synchrones (V4 : REST/OpenFeign) et asynchrones (V5 : RabbitMQ, V7 : Kafka).
- Documenter les références distantes, les invariants métier et les règles de résilience.
- Préparer les prérequis et les instructions pour les jalons suivants. **V0 ne nécessite pas encore d'application exécutable.**

## 3. Microservices
| Service | Responsabilité | Entités | Base dédiée (proposée) |
|---|---|---|---|
| `service-films` | Catalogue et séances, disponibilité des places | `Movie`, `Projection` | `films_db` |
| `service-utilisateurs` | Spectateurs, email unique, rôle, état actif | `User` | `users_db` |
| `service-reservations` | Réservations, états, QR code | `Reservation` | `reservations_db` |
| `service-paiements` | Encaissements et remboursements | `Payment` | `payments_db` |
| `service-notification` (V5+) | Envoi simulé et historique facultatif | `Notification` (facultatif) | `notifications_db` (facultative) |
| `api-gateway` | Routage, limitation des requêtes (V6) | Aucune | Aucune |

Chaque service est propriétaire de sa base. Pas de clé étrangère SQL ni d'association JPA entre bases de services.

## 4. Schéma d'architecture
Voir [`docs/architecture.md`](docs/architecture.md) et [`docs/architecture.mmd`](docs/architecture.mmd).

## 5. Diagrammes de classes
Voir [`docs/classes.md`](docs/classes.md). Les classes `*Externe` sont des représentations UML logiques, **pas des entités JPA**.

### Références et validation
- `Projection.filmId` : référence **locale** à `Movie`, validée dans `service-films`.
- `Reservation.userId` : référence **distante** à `User`, identifiée par ID, sans `@ManyToOne`.
- `Reservation.projectionId` : référence **distante** à `Projection`, identifiée par ID, sans `@ManyToOne`.
- `Payment.reservationId` : référence **distante** à `Reservation`, identifiée par ID, sans `@ManyToOne`.
- En V4, toute création/modification de référence distante exige une validation OpenFeign ; refuser explicitement en cas d'absence ou d'indisponibilité du service distant, sans sauvegarder.
- Les utilisateurs référencés doivent exister, être actifs et avoir le rôle `SPECTATEUR`.
- Ne jamais modifier librement `placesRestantes`, `montant` ou `statut` : utiliser des opérations métier atomiques.
- Les suppressions incompatibles avec des réservations ou l'historique sont refusées ; préférer une désactivation ou annulation métier.

## 6. Flux métier V4 — synchrone
1. Client -> Gateway -> `service-reservations` : demande de réservation.
2. `service-reservations` -> `service-utilisateurs` (OpenFeign) : valider utilisateur actif et rôle.
3. `service-reservations` -> `service-films` (OpenFeign) : vérifier la séance et réserver atomiquement N places pour l'opération.
4. `service-reservations` -> `service-paiements` (OpenFeign) : demander l'encaissement.
5. `service-paiements` -> `service-reservations` (OpenFeign) : vérifier l'existence de la réservation.
6. Succès : confirmer et générer le QR code. Échec : annuler, libérer les places ; rembourser tout paiement effectivement encaissé.
7. Consultation enrichie : `service-reservations` agrège utilisateur, film/séance et paiement sans fusionner les bases.

Les requêtes métier doivent porter un identifiant d'opération afin d'éviter les doubles réservations et doubles paiements.

## 7. V5 — RabbitMQ
Après validation de leurs opérations, les services métier publient les événements de confirmation, d'annulation de séance et de remboursement. `service-notification` est le **seul** service qui envoie les notifications simulées (email ou affichage). Les messages contiennent une adresse email ou une référence permettant de la résoudre. Destinataires administratifs configurables.

## 8. V7 — Kafka et transactions distribuées
**Créer V7 depuis la branche V3**, pas en migrant directement le code V5. Remplacer RabbitMQ pour les notifications. Alimenter des projections locales des référentiels par événements de création, modification, désactivation/suppression ; prévoir initialisation et reconstruction.

Événements distincts : `reservation-requested`, `seats-reserved`, `seats-rejected`, `payment-requested`, `payment-succeeded`, `payment-failed`, `reservation-confirmed`, `screening-cancelled`, `reservation-cancelled`, `payment-refunded`.

Format minimal des événements : `eventId`, `correlationId`, `aggregateId` (identifiant métier), `version`, `occurredAt`, `payload`.

Garanties à implémenter :
- Outbox transactionnelle pour publication fiable après commit local, avec reprise.
- Inbox/registre des événements traités pour déduplication et idempotence.
- Contrôles atomiques sur places disponibles et paiements, avec concurrence maîtrisée par le service propriétaire.
- Suivi des réservations de places par identifiant d'opération, pour ne libérer qu'une fois les places réellement retenues.
- Gestion des versions et des messages désordonnés.
- Délais d'attente, refus et compensations ; un paiement tardif après annulation doit être remboursé, jamais reconfirmer la réservation.
- Gestion globale des erreurs et Rate Limiting de Gateway (V6) réintroduits.

## 9. Stack technique proposée
- Java 17 ou 21 (choisir une version compatible avec Spring Boot et Spring Cloud retenus).
- Spring Boot, Spring Web, Spring Data JPA, Bean Validation.
- Spring Cloud OpenFeign (V4), Spring Cloud Gateway (V6).
- PostgreSQL : une base logique par service métier.
- RabbitMQ (V5) et Apache Kafka (V7).
- Maven, Git, Docker / Docker Compose, Postman ou Bruno.
- Mermaid (diagrammes dans Markdown), IDE IntelliJ IDEA ou VS Code.

Les versions exactes des dépendances doivent être fixées selon la matrice officielle de compatibilité Spring Boot / Spring Cloud avant le développement.

## 10. Prérequis
1. Installer un JDK compatible (17 ou 21), puis vérifier : `java -version`.
2. Installer Maven (ou utiliser le wrapper Maven des futurs services) : `mvn -version`.
3. Installer Git : `git --version`.
4. Installer Docker Desktop avec Docker Compose : `docker --version` et `docker compose version`.
5. Installer un IDE Java et un client API (Postman ou Bruno).
6. Installer une extension Mermaid ou consulter les fichiers `.md` dans un lecteur compatible.

## 11. Instructions d'utilisation de V0
```bash
git clone <URL_DU_DEPOT>
cd cine-resa
```
Ouvrir `README.md`, puis `docs/architecture.md` et `docs/classes.md` dans VS Code. Le fichier `docs/architecture.mmd` peut être importé dans un éditeur Mermaid. **Aucun `docker compose up` ou `mvn spring-boot:run` n'est possible en V0**, puisque le code des services et le fichier Compose ne sont pas encore livrés.

## 12. Structure cible du dépôt
```text
cine-resa/
├── README.md
├── docs/
│   ├── architecture.md
│   ├── architecture.mmd
│   └── classes.md
├── api-gateway/                 # à créer plus tard
├── service-films/               # V1
├── service-utilisateurs/        # V1
├── service-reservations/        # V4
├── service-paiements/           # V4
├── service-notification/        # V5
└── docker-compose.yml           # futur jalon
```

## 13. Exemples d'API prévues (indicatives)
- `GET /api/movies`, `POST /api/movies`, `GET /api/projections`
- `GET /api/users/{id}`, `POST /api/users`
- `POST /api/reservations`, `GET /api/reservations/{id}/details`, `POST /api/reservations/{id}/cancel`
- `GET /api/payments/{id}` ; encaissement/remboursement via opérations métier dédiées.

Ces chemins sont des propositions de conception, non des endpoints déjà implémentés.

## 14. Critères de validation du jalon V0
- [ ] Architecture globale avec Gateway, 4 services métier, notification, bases et brokers.
- [ ] Responsabilité unique de chaque microservice explicitée.
- [ ] Diagramme de classes par service, attributs et relations corrects.
- [ ] Références locales et distantes clairement distinguées.
- [ ] Flux OpenFeign, RabbitMQ et Kafka expliqués selon les jalons.
- [ ] Règles de cohérence, idempotence, compensation et résilience documentées.
- [ ] README complet avec prérequis, structure et instructions.

## 15. Choix de conception / limites
- La notification et son stockage sont facultatifs en V0, mais le service sera requis en V5.
- L'authentification n'est pas imposée ; l'identité du spectateur est représentée par `userId`.
- Le plan de salle Angular est un bonus facultatif.
- La simulation du paiement ne constitue pas une intégration bancaire réelle.

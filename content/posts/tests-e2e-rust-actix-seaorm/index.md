---
title: "Tests E2E en Rust avec Actix-web, SeaORM et Testcontainers"
date: 2026-04-07
tags: [rust, testing, actix-web, seaorm]
type: post
showTableOfContents: true
---

Je partage ici un retour d'expérience sur la mise en place de tests end-to-end dans une API Rust utilisant l'architecture hexagonale. Le stack : Actix-web pour le HTTP, SeaORM comme ORM, et Testcontainers pour avoir un vrai PostgreSQL en test.

Le chemin n'a pas été sans embûches. Entre les problèmes de version PostgreSQL, les pools de connexions qui fuient entre les runtimes Tokio, et les tests concurrents qui polluent la base de données, il y a eu pas mal de pièges à éviter.

## L'architecture cible

L'idée est simple : nos tests e2e traversent toute la stack, du HTTP jusqu'à PostgreSQL. Pas de mock de la base de données.

```
HTTP Request → Actix Controller → Service → Repository → PostgreSQL (testcontainers)
```

On utilise l'architecture hexagonale (ports & adapters). Le domaine définit un trait `TodoRepository`, et l'infrastructure fournit une implémentation PostgreSQL via SeaORM.

## La migration SQL

Notre schéma utilise un namespace dédié (`app`) plutôt que `public`, ce qui est courant en production pour isoler les données métier :

```sql
CREATE SCHEMA IF NOT EXISTS app;

CREATE TABLE app.todo (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    title       TEXT NOT NULL,
    completed   BOOLEAN NOT NULL DEFAULT FALSE,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Piège n°1 : la version de PostgreSQL

Le module `testcontainers-modules` pour Postgres utilise par défaut l'image `postgres:11-alpine`. C'est codé en dur dans le crate :

```rust
const TAG: &str = "11-alpine";
```

PostgreSQL 11, c'est 2018. Si votre migration utilise des fonctions comme `gen_random_uuid()` (PG 13+) ou `uuidv7()` (PG 18+), les migrations échoueront silencieusement avec :

```
function gen_random_uuid() does not exist
```

La solution : spécifier explicitement la version via `ImageExt::with_tag` :

```rust
use testcontainers::ImageExt;

let container = Postgres::default()
    .with_tag("16-alpine")
    .start()
    .await
    .expect("Failed to start PostgreSQL container");
```

## Piège n°2 : le pool de connexions qui fuit entre les runtimes

Chaque `#[tokio::test]` (ou `#[actix_web::test]`) crée son **propre runtime Tokio**. Le pool de connexions sqlx (utilisé par SeaORM) lance des tâches de maintenance en arrière-plan sur le runtime qui l'a créé.

Voici ce qui se passe si on partage un pool entre tests via un `OnceCell` :

1. Test 1 démarre → son runtime crée le pool → les tâches de maintenance tournent
2. Test 1 termine → **son runtime est détruit** → les tâches de maintenance meurent
3. Test 2 démarre → essaie d'utiliser le pool → les connexions rendues ne sont plus recyclées
4. → `ConnectionAcquire(Timeout)` après quelques tests

C'est un problème subtil car les 1 ou 2 premiers tests passent, puis les suivants échouent systématiquement.

### La solution : un pool frais par test

L'astuce est de séparer deux choses :

- Le **conteneur PostgreSQL** : partagé via `OnceCell`, démarré une seule fois
- Le **pool de connexions** : créé frais pour chaque test, sur le runtime de ce test

```rust
struct SharedPostgresContainer {
    _container: ContainerAsync<Postgres>,
    host_port: u16,
}

static SHARED_POSTGRES: OnceCell<Arc<SharedPostgresContainer>> = OnceCell::const_new();
```

Les migrations tournent sur une **connexion jetable**, fermée immédiatement après :

```rust
async fn get_shared_postgres() -> Arc<SharedPostgresContainer> {
    SHARED_POSTGRES
        .get_or_init(|| async {
            let container = Postgres::default()
                .with_tag("16-alpine")
                .start()
                .await
                .expect("Failed to start PostgreSQL container");

            let host_port = container
                .get_host_port_ipv4(5432)
                .await
                .expect("Failed to get PostgreSQL port");

            // Connexion jetable pour les migrations — pas liée au pool des tests
            let migration_url = format!(
                "postgres://postgres:postgres@127.0.0.1:{host_port}/postgres"
            );
            let mut migration_opts = ConnectOptions::new(migration_url);
            migration_opts.set_schema_search_path("app".to_string());
            let migration_conn = Database::connect(migration_opts)
                .await
                .expect("Failed to create migration connection");

            // Créer le schéma avant Migrator::up pour que seaql_migrations
            // atterrisse dans le schéma applicatif (pas dans public)
            migration_conn
                .execute_unprepared("CREATE SCHEMA IF NOT EXISTS app")
                .await
                .expect("Failed to create app schema");

            Migrator::up(&migration_conn, None)
                .await
                .expect("Failed to run migrations");

            migration_conn.close().await.expect("Failed to close migration connection");

            Arc::new(SharedPostgresContainer { _container: container, host_port })
        })
        .await
        .clone()
}
```

Puis chaque test reçoit son propre `ConnectionManager` :

```rust
pub async fn setup_test_database() -> (PostgresContainerHandle, Arc<ConnectionManager>) {
    let shared = get_shared_postgres().await;
    let conn_mgr = ConnectionManager::new(
        "127.0.0.1", shared.host_port,
        "postgres", "postgres", "postgres", "app",
    ).await;

    (PostgresContainerHandle { _shared: shared.clone() }, Arc::new(conn_mgr))
}
```

## Piège n°3 : le search_path et les schémas non-public

Quand on utilise un schéma autre que `public`, il faut configurer le `search_path` au niveau de la connexion SeaORM :

```rust
let mut opts = ConnectOptions::new(url);
opts.set_schema_search_path(schema.to_string());
```

Cela permet à SeaORM de trouver les tables dans le bon schéma. La table de métadonnées `seaql_migrations` atterrira elle aussi dans ce schéma, à condition de le créer **avant** d'appeler `Migrator::up` (voir le code des fixtures plus haut).

## Piège n°4 : les tests concurrents

Par défaut, `cargo test` lance les tests en parallèle. Si votre endpoint traite des données globalement (par exemple un `compute_interest` qui parcourt tous les prêts actifs), deux tests concurrents vont se marcher dessus.

En CI, on a observé :
- Un test attendait 3 résultats, en a trouvé 4 (les données de l'autre test)
- Un autre a reçu une erreur 500 (violation de contrainte d'unicité)

La solution est `serial_test`. C'est un crate qui fournit un attribut `#[serial]` pour sérialiser certains tests :

```rust
use serial_test::serial;

#[actix_web::test]
#[serial]
async fn test_create_todo_via_http() {
    // Ce test ne tournera jamais en même temps qu'un autre test marqué #[serial]
}
```

On l'ajoute aux `dev-dependencies` :

```toml
[dev-dependencies]
serial_test = "3"
```

## Le test e2e complet

Voici un test complet qui traverse toute la stack HTTP → Controller → Service → Repository → PostgreSQL :

```rust
#[actix_web::test]
#[serial]
async fn test_create_todo_via_http() {
    // GIVEN une app actix branchée sur testcontainers
    let (_handle, conn_mgr) = setup_test_database().await;

    let repo = PostgresTodoRepository::new(
        conn_mgr.master().clone(),
        conn_mgr.replica().clone(),
    );
    let app = test::init_service(
        App::new()
            .app_data(web::Data::new(AppState {
                todo_repo: Arc::new(repo),
            }))
            .configure(controller::configure_routes),
    )
    .await;

    // WHEN on crée un todo via POST
    let req = test::TestRequest::post()
        .uri("/api/todos")
        .set_json(CreateTodo { title: "Write blog post".to_string() })
        .to_request();
    let resp = test::call_service(&app, req).await;

    // THEN on reçoit 201 Created
    assert_eq!(resp.status(), 201);
    let body: Todo = test::read_body_json(resp).await;
    assert_eq!(body.title, "Write blog post");
    assert!(!body.completed);
}
```

Points importants :

- L'app actix est construite **inline** dans chaque test. Tenter de la retourner depuis une fonction helper pose des problèmes de types avec `actix_http::Request` (le type concret n'est pas ré-exporté par `actix_web`).
- Le seeding des données de test se fait par insertion directe en base. Ce sont des **préconditions**, pas la chose testée.
- Les assertions passent par les endpoints HTTP : on vérifie le status code et le body JSON.

## Le feature flag

Les tests e2e nécessitent Docker. On les isole derrière un feature flag pour ne pas les lancer par défaut :

```rust
// tests/e2e_tests.rs
#![cfg(feature = "e2e_tests")]
mod e2e;
```

```toml
# Cargo.toml
[features]
e2e_tests = []
```

```bash
# Lancer uniquement les tests e2e
cargo test --features e2e_tests --test e2e_tests

# Lancer tous les tests (unitaires + e2e)
cargo test --features e2e_tests
```

## Récapitulatif

| Piège | Symptôme | Solution |
|-------|----------|----------|
| Version PG trop vieille | `function xyz() does not exist` | `.with_tag("16-alpine")` |
| Pool partagé entre runtimes | `ConnectionAcquire(Timeout)` après 2-3 tests | Pool frais par test, conteneur partagé |
| Schéma non-public | `relation "xxx" does not exist` | `set_schema_search_path("app")` + créer le schéma avant `Migrator::up` |
| Tests concurrents | Données polluées, violations d'unicité | `#[serial]` via `serial_test` |

Chacun de ces problèmes est silencieux : les premiers tests passent souvent, c'est les suivants qui cassent. En CI c'est encore pire car les tests tournent en parallèle par défaut.

## Les limites de cette approche

Cette solution fonctionne bien pour une suite de tests e2e de taille raisonnable, mais il faut être conscient de ses limites.

Tous les tests marqués `#[serial]` s'exécutent les uns après les autres. Avec 5 tests qui prennent chacun 1 seconde, ça passe. Avec 50 tests, on commence à sentir le poids. Et avec 200, la CI devient un goulot d'étranglement.

Le problème de fond est qu'on a une seule base de données partagée. Dès qu'un endpoint manipule des données globalement (sans filtre par book, par tenant, etc.), deux tests concurrents se marchent dessus. `#[serial]` règle le problème en supprimant la concurrence, mais au prix du temps d'exécution.

Pour paralléliser, il faudrait isoler chaque test (ou groupe de tests) dans sa propre base. Quelques pistes :

- **Un conteneur PostgreSQL par test** : isolation parfaite, mais le coût de démarrage d'un conteneur (~1-2s) multiplié par le nombre de tests devient vite prohibitif
- **Une base de données par test** (même conteneur, `CREATE DATABASE` distinct) : plus léger qu'un conteneur, mais il faut quand même jouer les migrations pour chaque base
- **Un schéma par test** : encore plus léger, mais nécessite de paramétrer dynamiquement le `search_path` et le nom du schéma dans les migrations

Chacune de ces approches ajoute de la complexité. Pour un projet avec une dizaine de tests e2e, la solution présentée ici est largement suffisante. Au-delà, il faudra investir dans une infrastructure de test plus sophistiquée.

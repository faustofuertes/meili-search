# Configurar API key en Meilisearch

Hay dos niveles:

1. **Master key** – La clave principal que protege toda la instancia. Sin ella (en desarrollo) todas las rutas son públicas excepto `/keys`.
2. **API keys** – Claves adicionales que creas con la master key; cada una con permisos limitados (acciones e índices).

---

## 1. Configurar la Master Key

La master key tiene que tener **al menos 16 caracteres**. En **production** es obligatoria.

### Opción A: Variable de entorno

```bash
export MEILI_MASTER_KEY="tu_clave_secreta_minimo_16_chars"
cargo run --release
```

### Opción B: Archivo `config.toml`

En la raíz del repo, edita `config.toml` y descomenta/ajusta la línea:

```toml
master_key = "tu_clave_secreta_minimo_16_chars"
```

Luego arranca Meilisearch como siempre; leerá la clave del config.

### Opción C: Argumento al arrancar

```bash
./target/release/meilisearch --master-key "tu_clave_secreta_minimo_16_chars"
```

O con cargo:

```bash
cargo run --release -- --master-key "tu_clave_secreta_minimo_16_chars"
```

**Prioridad:** argumento > variable de entorno > `config.toml`.

### ¿Y el `.env`? ¿Y en Fly?

- **Meilisearch no lee archivos `.env`**; solo variables de entorno. Si usás `.env` en local, tenés que cargarlo antes de arrancar (`set -a && source .env && set +a`). Es solo comodidad para no escribir la clave en la terminal.
- **En Fly (o cualquier cloud)** el `.env` no se sube (está en `.gitignore`). Ahí la forma correcta es usar **secrets de la plataforma**: en Fly, `fly secrets set MEILI_MASTER_KEY="tu_clave"`. Fly inyecta eso como variable de entorno al arrancar y Meilisearch la lee igual. No hace falta (ni conviene) subir un `.env` al servidor.

**Recomendación:** En local, variable de entorno (o `config.toml`) es más estándar; `.env` sirve si querés tener la clave en un archivo y no exportarla a mano. En producción, siempre usar los secrets de la plataforma (Fly, etc.).

---

## 2. Usar la API key en las peticiones

Una vez configurada la master key, todas las peticiones (excepto `GET /health`) deben enviar el header:

```
Authorization: Bearer <tu_master_key>
```

Ejemplo con curl:

```bash
curl -H "Authorization: Bearer tu_clave_secreta_minimo_16_chars" \
  'http://localhost:7700/indexes'
```

```bash
curl -X POST 'http://localhost:7700/indexes/peliculas/documents' \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer tu_clave_secreta_minimo_16_chars' \
  --data-binary @ejemplo-peliculas.json
```

Sin ese header (o con una key inválida), recibirás `401` o `403`.

---

## 3. Crear API keys adicionales (opcional)

Con la master key puedes crear otras keys con permisos limitados (solo búsqueda, solo un índice, etc.). La ruta `/keys` **solo funciona si hay master key configurada**.

### Crear una key

```bash
curl -X POST 'http://localhost:7700/keys' \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer TU_MASTER_KEY' \
  -d '{
    "description": "Key solo búsqueda en películas",
    "actions": ["search"],
    "indexes": ["peliculas"]
  }'
```

- **actions**: lista de permisos. Ejemplos: `["search"]`, `["documents.add"]`, `["*"]` (todo).
- **indexes**: en qué índices aplica. `["*"]` = todos; `["peliculas", "products"]` = solo esos.

La respuesta incluye el campo **`key`**: esa es la clave en texto plano. **Solo se devuelve una vez**; guárdala.

### Listar keys

```bash
curl -H 'Authorization: Bearer TU_MASTER_KEY' 'http://localhost:7700/keys'
```

### Obtener una key por uid (sin revelar el valor)

```bash
curl -H 'Authorization: Bearer TU_MASTER_KEY' 'http://localhost:7700/keys/<key_uid>'
```

---

## Resumen

| Qué quieres              | Cómo                          |
|--------------------------|--------------------------------|
| Proteger la instancia    | Definir `master_key` (env, config o `--master-key`) |
| Llamar a la API          | Header `Authorization: Bearer <key>` |
| Keys con menos permisos  | Con master key: `POST /keys` con `actions` e `indexes` |

Documentación oficial: [Master key](https://www.meilisearch.com/docs/learn/security/master_api_keys), [API keys](https://www.meilisearch.com/docs/learn/security/api_keys).

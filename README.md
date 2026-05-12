# Programowanie Aplikacji Geoinformatycznych - Szablon Projektu

Gotowy do uruchomienia szablon aplikacji geoinformatycznej oparty na otwartych technologiach.
Studenci rozwijaja projekt, podmieniajac dane przykladowe na wlasne i rozszerzajac kod.

## Technologie

| Technologia | Rola |
|-------------|------|
| **FastAPI** | REST API backendu |
| **PostGIS** | Baza danych przestrzennych (dane wektorowe) |
| **pgSTAC** | Rozszerzenie PostgreSQL do katalogu STAC |
| **stac-fastapi** | Serwer STAC API (OGC STAC 1.0) |
| **TiTiler** | Serwowanie kafli rastrowych z plikow COG |
| **tipg** | Serwowanie kafli wektorowych MVT z PostGIS |
| **Streamlit** | Interfejs uzytkownika z mapa MapLibre GL |
| **Docker + UV** | Konteneryzacja i zarzadzanie srodowiskiem Python |

## Architektura

```
Przegladarka
    |
    +-- MapLibre JS (kafle MVT) --> tipg :8008 --> PostGIS
    +-- MapLibre JS (kafle XYZ) --> TiTiler :7800 --> COG (URL)
    |
Streamlit :8501
    |
    +-- httpx --> FastAPI :8000 --> PostGIS
    +-- httpx --> STAC API :8080 --> PostGIS (pgSTAC)
```

## Wymagania

- [Docker Desktop](https://www.docker.com/products/docker-desktop/)
- [Make](https://www.gnu.org/software/make/) (Windows: `choco install make` lub Git Bash)

## Szybki start

```bash
# 1. Sklonuj repozytorium
git clone <url>
cd python1-project

# 2. Skopiuj plik konfiguracyjny
cp .env.example .env

# 3. Zbuduj obrazy
make build

# 4. Uruchom serwisy i zaladuj dane przykladowe
make init

# 5. Otworz aplikacje
# Streamlit:  http://localhost:8501
# FastAPI:    http://localhost:8000/docs
# STAC API:   http://localhost:8080
# TiTiler:    http://localhost:7800/docs
# tipg:       http://localhost:8008
```

> **Uwaga:** `make init` uruchamia serwisy i jednorazowo wykonuje skrypt `db-init`, ktory laduje dane przykladowe do bazy.

## Dostepne komendy Make

```bash
make up               # Uruchom serwisy w tle
make down             # Zatrzymaj serwisy
make build            # Zbuduj obrazy Dockera
make init             # Uruchom serwisy i zaladuj dane (pierwsze uruchomienie)
make logs             # Sledz logi (Ctrl+C aby wyjsc)
make ps               # Status serwisow
make reset            # Zatrzymaj i usun dane (volumes) - UWAGA: kasuje baze
make restart s=<serwis>  # Zrestartuj konkretny serwis (np. make restart s=frontend)
make shell s=<serwis>    # Terminal w kontenerze (np. make shell s=backend)
make shell-db         # Otworz powloke psql
```

## Struktura projektu

```
python1-project/
|
|- docker-compose.yml          # Definicja wszystkich serwisow
|- .env.example                # Przykladowe zmienne srodowiskowe
|- Makefile                    # Skroty do czesto uzywanych komend
|
|- data/                       # Dane przykladowe (zastap swoimi!)
|   |- sample_features.geojson # Punkty Krakowa -> PostGIS
|   |- stac_collection.json    # Definicja kolekcji STAC
|   +- stac_items.json         # Itemy STAC z linkami do COG
|
|- scripts/                    # Jednorazowe ladowanie danych (db-init)
|   |- ingest_data.py          # Glowny skrypt ingestion
|   |- Dockerfile
|   +- pyproject.toml
|
|- backend/                    # FastAPI - tu piszesz logike biznesowa
|   |- src/app/
|   |   |- main.py             # Konfiguracja aplikacji
|   |   |- config.py           # Zmienne srodowiskowe (pydantic-settings)
|   |   |- database.py         # Klasa Database (asyncpg)
|   |   |- models.py           # Modele danych (Pydantic)
|   |   |- dependencies.py     # Dependency injection (get_db)
|   |   +- routers/
|   |       +- locations.py    # Endpointy /locations/
|   |- Dockerfile
|   +- pyproject.toml
|
|- frontend/                   # Streamlit - tu piszesz UI
|   |- src/
|   |   |- app.py              # Glowna aplikacja Streamlit
|   |   +- config.py           # Adresy serwisow
|   |- Dockerfile
|   +- pyproject.toml
|
|- stac-api/                   # stac-fastapi-pgstac (minimalna konfiguracja)
|   |- main.py
|   |- Dockerfile
|   +- pyproject.toml
|
+- tipg/                       # tipg - serwer kafli wektorowych
    |- Dockerfile
    +- pyproject.toml
```

## Jak rozszerzyc projekt

### 1. Podmien dane

Edytuj lub zastap pliki w `data/`:
- `sample_features.geojson` - twoje dane wektorowe (GeoJSON)
- `stac_collection.json` - definicja kolekcji STAC
- `stac_items.json` - linki do twoich plikow COG

Zmodyfikuj `scripts/ingest_data.py` - klasa `DataIngester`:
- `_setup_tables()` - schemat tabel
- `_load_vector_data()` - ladowanie danych wektorowych
- `_load_stac_catalog()` - ladowanie katalogu STAC

Przeladuj dane:
```bash
docker compose run --rm db-init
```

### 2. Dodaj nowy endpoint w FastAPI

Stwórz plik `backend/src/app/routers/moj_router.py`:

```python
from fastapi import APIRouter, Depends
from app.database import Database
from app.dependencies import get_db

router = APIRouter()

@router.get("/moje-dane")
async def get_moje_dane(db: Database = Depends(get_db)) -> list:
    rows = await db.fetch("SELECT * FROM app.moja_tabela")
    return [dict(row) for row in rows]
```

Zarejestruj router w `backend/src/app/main.py`:
```python
from app.routers import locations, moj_router
app.include_router(moj_router.router, prefix="/moje-dane", tags=["moje-dane"])
```

Dzieki `--reload` zmiany sa widoczne natychmiast (bez przebudowy obrazu).

### 3. Rozszerz mape w Streamlit

Edytuj `frontend/src/app.py`. Przyklad dodania nowej warstwy wektorowej:

```python
m.add_source("moja-warstwa", VectorTileSource(
    tiles=[f"{settings.public_tipg_url}/collections/app.moja_tabela/tiles/WebMercatorQuad/{{z}}/{{x}}/{{y}}"]
))
m.add_layer(Layer(
    type=LayerType.FILL,
    id="moja-warstwa-fill",
    source="moja-warstwa",
    source_layer="default",  # tipg zawsze uzywa nazwy "default" w MVT
    paint={"fill-color": "#4264fb", "fill-opacity": 0.5},
))
```

## Serwisy - szczegoly

### PostGIS + pgSTAC (`:5432`)
Baza danych PostgreSQL z rozszerzeniami PostGIS (dane przestrzenne) i pgSTAC (katalog STAC).
Schemat `app` przechowuje dane aplikacji. Schemat `pgstac` - dane katalogu STAC.

### STAC API (`:8080`)
Implementacja OGC STAC API 1.0. Endpointy:
- `GET /collections` - lista kolekcji
- `GET /collections/{id}/items` - lista itemow
- `POST /search` - wyszukiwanie przestrzenne i temporalne

### TiTiler (`:7800`)
Dynamiczny serwer kafli rastrowych dla plikow Cloud Optimized GeoTIFF (COG).
Przyklady: `/cog/info?url=...`, `/cog/tiles/{z}/{x}/{y}?url=...`

### tipg (`:8008`)
Serwer kafli wektorowych MVT bezposrednio z tabel PostGIS (schemat `app`).
Endpointy OGC API Features + Tiles. Tabele sa odkrywane automatycznie.

### FastAPI (`:8000`)
Backend REST API z automatyczna dokumentacja Swagger UI pod `/docs`.

### Streamlit (`:8501`)
Frontend z mapa MapLibre GL, przegladarka STAC i informacje o projekcie.

## Zmienne srodowiskowe

Plik `.env` (skopiowany z `.env.example`):

```
POSTGRES_USER=postgres      # uzytkownik bazy danych
POSTGRES_PASSWORD=postgres  # haslo
POSTGRES_DB=geoapp          # nazwa bazy
```

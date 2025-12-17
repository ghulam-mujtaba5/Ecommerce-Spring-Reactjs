# Copilot Instructions for E-Commerce Application

## Architecture Overview

This is a **full-stack e-commerce application** with a clear separation between backend and frontend, orchestrated via Docker Compose.

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│  React/Vite     │────▶│  Spring Boot    │────▶│  PostgreSQL     │
│  (port 80)      │     │  (port 5000)    │     │  (port 5432)    │
│  ecom-frontend/ │     │  ecom-backend/  │     │  Docker volume  │
└─────────────────┘     └─────────────────┘     └─────────────────┘
```

- **Backend**: Spring Boot 3.4 REST API at `/api/v1/*` with JWT authentication
- **Frontend**: React 19 SPA using Redux Toolkit for state, Axios for API calls
- **Database**: PostgreSQL 15 (H2 available for local dev without Docker)

## Key Development Commands

```bash
# Backend (from ecom-backend/)
mvn clean package -DskipTests     # Build JAR
mvn spring-boot:run               # Run locally (needs DB)

# Frontend (from ecom-frontend/)
npm ci && npm run dev             # Dev server with HMR
npm run build                     # Production build
npm run lint                      # ESLint check

# Full stack with Docker
docker-compose up --build         # Start all services
docker-compose down -v            # Stop and clean volumes
```

## Backend Patterns (`ecom-backend/`)

### Layer Structure
- **Controllers** (`controller/`): REST endpoints at `/api/v1/*`, use `@RestController`
- **Services** (`service/` + `service/impl/`): Interface + implementation pattern
- **Repositories** (`repositories/`): Spring Data JPA interfaces
- **DTOs** (`payload/`): Named `*DTO.java` and `*Response.java` for API responses
- **Exceptions** (`exceptions/`): Custom `ResourceNotFoundException`, `APIException`, handled by `GlobalExceptionHandler`

### Conventions
- Use `ModelMapper` for entity ↔ DTO conversion (configured in [AppConfig.java](ecom-backend/src/main/java/com/jianquanwang/ecommerce/config/AppConfig.java))
- Pagination defaults in [AppConstants.java](ecom-backend/src/main/java/com/jianquanwang/ecommerce/config/AppConstants.java): `PAGE_SIZE=50`, `SORT_DIR=asc`
- All list endpoints return paginated `*Response` wrapper objects with `content`, `pageNumber`, `totalPages`, etc.
- JWT auth configured in `security/` - `/api/v1/**` routes are currently `permitAll()` for dev

### Adding a New Entity
1. Create model in `model/` with JPA annotations + Lombok `@Data`
2. Create repository interface extending `JpaRepository`
3. Create DTO in `payload/` and Response wrapper if paginated
4. Create service interface + implementation in `service/impl/`
5. Create controller with RESTful endpoints

## Frontend Patterns (`ecom-frontend/`)

### State Management (Redux)
- Store configured in [store/reducers/store.js](ecom-frontend/src/store/reducers/store.js)
- Actions in [store/actions/index.js](ecom-frontend/src/store/actions/index.js) - use thunks with `dispatch`
- Reducers use string action types like `"FETCH PRODUCTS"`, `"IS LOADING"`, `"IS ERROR"`

### API Communication
- Base Axios instance in [api/api.js](ecom-frontend/src/api/api.js)
- Uses `VITE_BACK_END_URL` env var for backend URL
- All API calls go through centralized Axios instance

### Component Structure
- Components in `src/components/` - functional components with hooks
- Styling: Tailwind CSS + MUI components
- Use `useSelector`/`useDispatch` from react-redux for state

## Environment Configuration

### Backend (`application.properties`)
```properties
server.port=5000
spring.datasource.url=jdbc:postgresql://localhost:5432/ecommerce
spring.app.jwtSecret=<min-50-chars>
```

### Frontend (`.env` or Docker)
```
VITE_BACK_END_URL=http://localhost:5000
```

### Docker Compose (production-like)
Environment variables override properties via `SPRING_DATASOURCE_*` naming convention.

## Testing

- Backend tests use `application-test.properties` with `spring.jpa.hibernate.ddl-auto=create-drop`
- Tests run against PostgreSQL in CI (GitHub Actions provides the service)
- Run backend tests: `mvn test -Dspring.profiles.active=test`

## CI/CD Pipeline

Defined in [.github/workflows/ci-cd.yml](.github/workflows/ci-cd.yml):
1. **Build**: Maven package + npm build
2. **Lint**: ESLint + npm audit
3. **Test**: Backend tests with PostgreSQL service
4. **Docker**: Multi-stage builds for both services
5. **Deploy**: Pushes to Docker Hub

Required GitHub Secrets: `DOCKERHUB_USERNAME`, `DOCKERHUB_TOKEN`

## Docker Notes

Both services use multi-stage Dockerfiles:
- **Backend**: `maven:3.9-eclipse-temurin-21` build → `eclipse-temurin:21-jre-alpine` runtime
- **Frontend**: `node:18-alpine` build → `nginx:alpine` serve static files

Health checks configured for all services in `docker-compose.yml`.

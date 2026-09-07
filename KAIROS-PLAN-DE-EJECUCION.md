# Kairós La Paz - Plan de Ejecución Paralela

**Objetivo:** Backend + Frontend en paralelo sin bloqueos.

**Duración:** 7 días iniciales (v0.1 - Base técnica)

**Enfoque:** Contratos OpenAPI → Código.

---

## Decisiones críticas a tomar HOY

### 1. Storage de JWT / Sesión

**Opción recomendada:** httpOnly cookies

```
Backend:
- Set-Cookie: session_id=...; HttpOnly; Secure; SameSite=Strict
- Frontend no puede leer la cookie
- Axios: withCredentials: true

Ventaja:
- XSS no expone token
- CSRF mitigado con SameSite
- No confiar en localStorage
```

**Decisión:** httpOnly cookies

---

### 2. State Management Stack

**Decisión recomendada:**

```
Context API     → Auth global
Zustand         → Carrito, filtros complejos
TanStack Query  → Datos remotos + caché
React state     → UI local
```

**No usar:**
- Redux (overkill para v0.1)
- MobX (complejidad innecesaria)
- Jotai (fragmentación)

**Decisión:**  Context + Zustand + TanStack Query

---

### 3. Versionado de API

```
/api/v1/

Cambios incompatibles futuros = /api/v2/
(no dentro de v1)
```

**Decisión:**  /api/v1/ inmutable durante v0.1-v1.0

---

### 4. Generación de tipos desde OpenAPI

```
Backend: FastAPI
         ↓
openapi.json
         ↓
openapi-typescript-codegen
         ↓
src/api/generated/
         ↓
types + cliente HTTP
```

**Herramienta:** `openapi-typescript` o `openapi-fetch`

**Decisión:**  Automatizar tipos

---

### 5. Base de datos

```
PostgreSQL 15+
Connection: psycopg (async)
Migrations: Alembic
```

**Decisión:**  Confirmado en docs

---

---

## Timeline por días

### Día 1 - Contratos + Setup

#### Backend (4 horas)

```bash
# 1. Repo + estructura
git clone / git init
mkdir app/{core,models,schemas,routers,services,repositories}

# 2. Dependencias
cat > requirements.txt << EOF
fastapi==0.104.1
uvicorn[standard]==0.24.0
sqlalchemy==2.0.23
psycopg[binary]==3.19.1
pydantic==2.5.0
pydantic-settings==2.1.0
python-jose[cryptography]==3.3.0
bcrypt==4.1.1
python-multipart==0.0.6
alembic==1.13.0
pytest==7.4.3
pytest-asyncio==0.21.1
httpx==0.25.2
EOF

pip install -r requirements.txt

# 3. OpenAPI schema (borrador)
touch openapi.json
```

**Definir primeros endpoints:**

```yaml
# openapi.json (simplificado)
openapi: 3.0.0
info:
  title: Kairós API
  version: 1.0.0

paths:
  /api/v1/auth/login:
    post:
      requestBody:
        content:
          application/json:
            schema:
              type: object
              properties:
                email: { type: string }
                password: { type: string }
      responses:
        '200':
          description: Login exitoso
          content:
            application/json:
              schema:
                type: object
                properties:
                  user: { $ref: '#/components/schemas/User' }
                  
  /api/v1/users/me:
    get:
      security:
        - cookieAuth: []
      responses:
        '200':
          content:
            application/json:
              schema: { $ref: '#/components/schemas/User' }
        '401':
          description: No autenticado

  /api/v1/camps:
    get:
      parameters:
        - name: page
          in: query
          schema: { type: integer, default: 1 }
        - name: page_size
          in: query
          schema: { type: integer, default: 20 }
      responses:
        '200':
          content:
            application/json:
              schema:
                type: object
                properties:
                  items: { type: array, items: { $ref: '#/components/schemas/Camp' } }
                  total: { type: integer }
                  page: { type: integer }
                  page_size: { type: integer }

components:
  schemas:
    User:
      type: object
      properties:
        id: { type: integer }
        email: { type: string }
        first_name: { type: string }
        last_name: { type: string }
        role: { type: string, enum: [INTEGRANTE, PARTICIPANTE, DIRECTOR_CAMPA] }
        created_at: { type: string, format: date-time }

    Camp:
      type: object
      properties:
        id: { type: integer }
        name: { type: string }
        description: { type: string, nullable: true }
        start_date: { type: string, format: date }
        end_date: { type: string, format: date }
        location: { type: string }
        capacity: { type: integer }
        cost: { type: number }
        status: { type: string, enum: [DRAFT, OPEN, CLOSED, IN_PROGRESS] }
        created_at: { type: string, format: date-time }
```

**Push a GitHub:**
```bash
git add .
git commit -m "chore: initial setup with openapi schema"
git push origin main
```

#### Frontend (4 horas)

```bash
# 1. Crear proyecto
npm create vite@latest kairos-frontend -- --template react-ts
cd kairos-frontend

# 2. Instalar deps principales
npm install axios zustand @tanstack/react-query react-router-dom

# 3. Crear .env
cat > .env << EOF
VITE_API_URL=http://localhost:8000/api/v1
EOF

# 4. Descargar OpenAPI y generar tipos
npm install -D openapi-typescript
npx openapi-typescript http://localhost:8000/openapi.json -o src/api/generated/types.ts

# 5. Estructura base
mkdir -p src/{components,pages,hooks,services,types,utils,context,store,styles}
```

**Crear cliente HTTP:**

```typescript
// src/api/client.ts
import axios from 'axios';

const api = axios.create({
  baseURL: import.meta.env.VITE_API_URL,
  timeout: 10000,
  withCredentials: true, // Enviar cookies
});

// Request interceptor
api.interceptors.request.use((config) => {
  config.headers['X-Request-ID'] = crypto.randomUUID();
  return config;
});

// Response interceptor
api.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      // Sesión expirada
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);

export default api;
```

**Crear Auth Context:**

```typescript
// src/context/AuthContext.tsx
import { createContext, useState, useEffect } from 'react';
import api from '../api/client';

interface User {
  id: number;
  email: string;
  first_name: string;
  last_name: string;
  role: string;
}

export const AuthContext = createContext<{
  user: User | null;
  loading: boolean;
  login: (email: string, password: string) => Promise<void>;
  logout: () => Promise<void>;
} | null>(null);

export function AuthProvider({ children }: { children: React.ReactNode }) {
  const [user, setUser] = useState<User | null>(null);
  const [loading, setLoading] = useState(true);

  // Al cargar, verificar si ya hay sesión
  useEffect(() => {
    api.get<{ user: User }>('/users/me')
      .then((res) => setUser(res.data.user))
      .catch(() => setUser(null))
      .finally(() => setLoading(false));
  }, []);

  const login = async (email: string, password: string) => {
    const res = await api.post<{ user: User }>('/auth/login', { email, password });
    setUser(res.data.user);
  };

  const logout = async () => {
    await api.post('/auth/logout');
    setUser(null);
  };

  return (
    <AuthContext.Provider value={{ user, loading, login, logout }}>
      {children}
    </AuthContext.Provider>
  );
}
```

**Push a GitHub:**
```bash
git add .
git commit -m "chore: initial setup with auth context and http client"
git push origin main
```

---

#### Entregable Día 1

✅ Backend: estructura + openapi.json borrador
✅ Frontend: estructura + cliente HTTP + Auth Context
✅ Ambos en GitHub

---

### Día 2 - Modelos + Schemas Backend

#### Backend (6 horas)

**Models (SQLAlchemy):**

```python
# app/models/user.py
from sqlalchemy import Column, Integer, String, DateTime, Enum as SQLEnum
from sqlalchemy.ext.declarative import declarative_base
from datetime import datetime
import enum

Base = declarative_base()

class UserRole(str, enum.Enum):
    INTEGRANTE = "INTEGRANTE"
    PARTICIPANTE = "PARTICIPANTE"
    DIRECTOR_CAMPA = "DIRECTOR_CAMPA"
    TESORERO = "TESORERO"
    COORDINADOR = "COORDINADOR"

class User(Base):
    __tablename__ = "users"

    id = Column(Integer, primary_key=True)
    email = Column(String, unique=True, index=True)
    first_name = Column(String)
    last_name = Column(String)
    password_hash = Column(String)
    role = Column(SQLEnum(UserRole), default=UserRole.INTEGRANTE)
    created_at = Column(DateTime, default=datetime.utcnow)
    updated_at = Column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)
```

**Schemas (Pydantic):**

```python
# app/schemas/user.py
from pydantic import BaseModel, EmailStr
from datetime import datetime
from enum import Enum

class UserRole(str, Enum):
    INTEGRANTE = "INTEGRANTE"
    PARTICIPANTE = "PARTICIPANTE"
    DIRECTOR_CAMPA = "DIRECTOR_CAMPA"

class UserCreate(BaseModel):
    email: EmailStr
    first_name: str
    last_name: str
    password: str

class UserResponse(BaseModel):
    id: int
    email: str
    first_name: str
    last_name: str
    role: UserRole
    created_at: datetime

    class Config:
        from_attributes = True
```

**Repetir para:** Camp, Registration, Commission, Inventory, Payment

**Commit:**
```bash
git add .
git commit -m "feat: add models and schemas for core entities"
```

---

### Día 3 - Rutas + Servicios Backend

#### Backend (6 horas)

**Services (lógica de negocio):**

```python
# app/services/auth_service.py
from app.schemas.user import UserCreate, UserResponse
from app.repositories.user_repository import UserRepository
from app.core.security import hash_password, verify_password
from fastapi import HTTPException

class AuthService:
    def __init__(self, repo: UserRepository):
        self.repo = repo

    def register(self, user_data: UserCreate) -> UserResponse:
        # Validar que no exista
        if self.repo.get_by_email(user_data.email):
            raise HTTPException(status_code=409, detail="Email already exists")
        
        # Crear
        user = self.repo.create(
            email=user_data.email,
            first_name=user_data.first_name,
            last_name=user_data.last_name,
            password_hash=hash_password(user_data.password)
        )
        return UserResponse.from_orm(user)

    def authenticate(self, email: str, password: str) -> UserResponse:
        user = self.repo.get_by_email(email)
        if not user or not verify_password(password, user.password_hash):
            raise HTTPException(status_code=401, detail="Invalid credentials")
        return UserResponse.from_orm(user)
```

**Routers:**

```python
# app/routers/auth.py
from fastapi import APIRouter, Depends, HTTPException
from fastapi.responses import JSONResponse
from app.schemas.user import UserCreate, UserResponse
from app.services.auth_service import AuthService
from app.core.security import create_access_token

router = APIRouter(prefix="/auth", tags=["auth"])

@router.post("/register", response_model=UserResponse)
def register(user_data: UserCreate, service: AuthService = Depends()):
    return service.register(user_data)

@router.post("/login")
def login(email: str, password: str, service: AuthService = Depends()):
    user = service.authenticate(email, password)
    
    # Crear session/JWT
    token = create_access_token({"sub": str(user.id)})
    
    response = JSONResponse({"user": user})
    response.set_cookie(
        key="session_id",
        value=token,
        httponly=True,
        secure=True,
        samesite="strict",
        max_age=30 * 60  # 30 min
    )
    return response

@router.get("/users/me", response_model=UserResponse)
def get_me(current_user: User = Depends(get_current_user)):
    return current_user
```

**Registrar en main.py:**

```python
# app/main.py
from fastapi import FastAPI
from app.routers import auth

app = FastAPI()

app.include_router(auth.router)

@app.get("/openapi.json")
def get_openapi():
    return app.openapi()
```

**Commit:**
```bash
git add .
git commit -m "feat: add auth service and routes with JWT/cookies"
```

---

### Día 4 - BD + Migraciones Backend

#### Backend (4 horas)

```bash
# 1. Inicializar Alembic
alembic init alembic

# 2. Configurar alembic.ini
# Actualizar DATABASE_URL

# 3. Crear primera migración
alembic revision --autogenerate -m "initial schema"

# 4. Aplicar
alembic upgrade head

# 5. Verificar BD
psql kairos_db -c "\dt"
```

**Resultado:** Tablas creadas en PostgreSQL

**Commit:**
```bash
git add .
git commit -m "feat: create initial database schema with alembic"
```

---

### Día 5 - Componentes Base + Rutas Frontend

#### Frontend (6 horas)

**Componentes:**

```typescript
// src/components/common/Button.tsx
interface ButtonProps {
  children: React.ReactNode;
  onClick?: () => void;
  loading?: boolean;
  variant?: 'primary' | 'secondary';
  disabled?: boolean;
}

export function Button({ children, loading, ...props }: ButtonProps) {
  return (
    <button disabled={loading || props.disabled} {...props}>
      {loading ? '...' : children}
    </button>
  );
}

// src/components/common/Input.tsx
interface InputProps {
  label: string;
  error?: string;
  value: string;
  onChange: (e: React.ChangeEvent<HTMLInputElement>) => void;
}

export function Input({ label, error, ...props }: InputProps) {
  return (
    <div>
      <label>{label}</label>
      <input {...props} />
      {error && <span style={{ color: 'red' }}>{error}</span>}
    </div>
  );
}

// src/components/common/LoadingSpinner.tsx
export function LoadingSpinner() {
  return <div>Cargando...</div>;
}

// src/components/common/ErrorMessage.tsx
export function ErrorMessage({ message }: { message: string }) {
  return <div style={{ color: 'red' }}>{message}</div>;
}
```

**Páginas:**

```typescript
// src/pages/auth/LoginPage.tsx
import { useState, useContext } from 'react';
import { useNavigate } from 'react-router-dom';
import { AuthContext } from '../../context/AuthContext';
import { Button, Input, ErrorMessage } from '../../components/common';

export function LoginPage() {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [error, setError] = useState('');
  const [loading, setLoading] = useState(false);
  const auth = useContext(AuthContext);
  const navigate = useNavigate();

  const handleLogin = async (e: React.FormEvent) => {
    e.preventDefault();
    setLoading(true);
    try {
      await auth!.login(email, password);
      navigate('/dashboard');
    } catch (err: any) {
      setError(err.response?.data?.detail || 'Error de autenticación');
    } finally {
      setLoading(false);
    }
  };

  return (
    <form onSubmit={handleLogin}>
      <h1>Iniciar sesión</h1>
      {error && <ErrorMessage message={error} />}
      <Input
        label="Email"
        type="email"
        value={email}
        onChange={(e) => setEmail(e.target.value)}
      />
      <Input
        label="Contraseña"
        type="password"
        value={password}
        onChange={(e) => setPassword(e.target.value)}
      />
      <Button loading={loading}>Iniciar sesión</Button>
    </form>
  );
}

// src/pages/public/HomePage.tsx
export function HomePage() {
  return <h1>Bienvenido a Kairós</h1>;
}
```

**Routing:**

```typescript
// src/routes/ProtectedRoute.tsx
import { Navigate } from 'react-router-dom';
import { useAuth } from '../hooks/useAuth';
import { LoadingSpinner } from '../components/common';

export function ProtectedRoute({ children }: { children: React.ReactNode }) {
  const { user, loading } = useAuth();

  if (loading) return <LoadingSpinner />;
  if (!user) return <Navigate to="/login" />;
  return <>{children}</>;
}

// src/App.tsx
import { BrowserRouter, Routes, Route } from 'react-router-dom';
import { AuthProvider } from './context/AuthContext';
import { LoginPage } from './pages/auth/LoginPage';
import { HomePage } from './pages/public/HomePage';
import { ProtectedRoute } from './routes/ProtectedRoute';

function App() {
  return (
    <AuthProvider>
      <BrowserRouter>
        <Routes>
          <Route path="/" element={<HomePage />} />
          <Route path="/login" element={<LoginPage />} />
          <Route
            path="/dashboard"
            element={
              <ProtectedRoute>
                <div>Dashboard</div>
              </ProtectedRoute>
            }
          />
        </Routes>
      </BrowserRouter>
    </AuthProvider>
  );
}

export default App;
```

**Commit:**
```bash
git add .
git commit -m "feat: add base components, pages, routing and auth flow"
```

---

### Día 6 - Integración + Testing

#### Backend (3 horas)

**Tests:**

```python
# tests/test_auth.py
import pytest
from fastapi.testclient import TestClient
from app.main import app

client = TestClient(app)

def test_register():
    response = client.post("/api/v1/auth/register", json={
        "email": "test@example.com",
        "first_name": "Test",
        "last_name": "User",
        "password": "securepass123"
    })
    assert response.status_code == 200
    assert response.json()["email"] == "test@example.com"

def test_login():
    # Register first
    client.post("/api/v1/auth/register", json={...})
    
    # Login
    response = client.post("/api/v1/auth/login", json={
        "email": "test@example.com",
        "password": "securepass123"
    })
    assert response.status_code == 200
    assert "session_id" in response.cookies

def test_get_me_unauthorized():
    response = client.get("/api/v1/users/me")
    assert response.status_code == 401

def test_get_me_authorized():
    # Login
    client.post("/api/v1/auth/login", json={...})
    
    # Get me
    response = client.get("/api/v1/users/me")
    assert response.status_code == 200
```

**Ejecutar:**
```bash
pytest -v
```

#### Frontend (3 horas)

**Tests:**

```typescript
// src/components/common/Button.test.tsx
import { render, screen } from '@testing-library/react';
import { Button } from './Button';

test('renders button with text', () => {
  render(<Button>Click me</Button>);
  expect(screen.getByText('Click me')).toBeInTheDocument();
});

test('shows loading state', () => {
  render(<Button loading>Click me</Button>);
  expect(screen.getByText('...')).toBeInTheDocument();
});
```

**Ejecutar:**
```bash
npm run test
```

---

#### Validar contrato OpenAPI

**Backend genera schema:**
```bash
curl http://localhost:8000/openapi.json > openapi.json
```

**Frontend genera tipos:**
```bash
npx openapi-typescript openapi.json -o src/api/generated/types.ts
```

**Verificar que tipos coinciden:**
```bash
npm run typecheck
```

---

**Commits:**
```bash
# Backend
git add .
git commit -m "test: add auth tests"

# Frontend
git add .
git commit -m "test: add component tests"
```

---

### Día 7 - Build + CI/CD

#### Backend (2 horas)

**GitHub Actions:**

```yaml
# .github/workflows/backend.yml
name: Backend CI

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_DB: kairos_db
          POSTGRES_PASSWORD: password
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.13'
      
      - name: Install dependencies
        run: |
          pip install -r requirements.txt
      
      - name: Lint
        run: |
          pip install flake8
          flake8 app/
      
      - name: Run tests
        run: |
          pytest -v
      
      - name: Build
        run: |
          echo "Build successful"
```

#### Frontend (2 horas)

**GitHub Actions:**

```yaml
# .github/workflows/frontend.yml
name: Frontend CI

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up Node
        uses: actions/setup-node@v3
        with:
          node-version: '18'
      
      - name: Install dependencies
        run: npm install
      
      - name: Lint
        run: npm run lint
      
      - name: Type check
        run: npm run typecheck
      
      - name: Run tests
        run: npm run test
      
      - name: Build
        run: npm run build
```

---

---

## Estado de v0.1 al final de Día 7

### ✅ Backend

```
[x] Estructura modular
[x] SQLAlchemy + PostgreSQL
[x] Models base (User, Camp, Registration, etc.)
[x] Schemas Pydantic
[x] Auth service (register, login)
[x] Auth routes
[x] JWT + httpOnly cookies
[x] Tests básicos
[x] Migraciones Alembic
[x] CI/CD con GitHub Actions
[x] OpenAPI schema generado
```

### ✅ Frontend

```
[x] Vite + React + TypeScript
[x] Axios client con interceptors
[x] Context API (Auth)
[x] Componentes base
[x] Rutas protegidas
[x] Login page
[x] Auth flow completo
[x] Tests básicos
[x] CI/CD con GitHub Actions
[x] Tipos generados desde OpenAPI
```

### ✅ Integración

```
[x] Frontend consume /auth/login
[x] Frontend almacena sesión en cookies
[x] Frontend obtiene /users/me
[x] Tipos sincronizados
[x] Ambos con CI/CD
[x] Documentación completa
```

---

---

## Próximas fases (después de Día 7)

### v0.2 - Usuarios + Roles (3 días)

```
Backend:
- Completar CRUD de usuarios
- Roles y permisos
- Middleware de autenticación

Frontend:
- Perfil de usuario
- Actualizar información
- Admin: gestionar usuarios
```

### v0.3 - Campamentos (5 días)

```
Backend:
- CRUD de campamentos
- Estados
- Capacidad y cupo

Frontend:
- Listar campamentos
- Detalle
- Crear (solo admin)
```

### v0.4 - Inscripciones (4 días)

```
Backend:
- Crear inscripción
- Validar cupo
- Formularios dinámicos

Frontend:
- Formulario de registro
- Estados de inscripción
```

---

---

## Principios durante desarrollo

### 1. Contrato primero

```
Cuando backend necesita nuevo endpoint:
  - Agregar a openapi.json
  - Generar tipos en frontend
  - Frontend implementa cliente
  - Backend implementa servicio
```

### 2. No bloquear

Si backend no está listo:
```
Frontend usa mock:
const camps = [{id: 1, name: "Test", ...}];

Se reemplaza cuando API esté lista.
```

### 3. Tests desde día 1

```
Cada feature:
- Code
- Test
- Commit
```

### 4. PR reviews

```
Antes de merge:
- CI pasa
- Tests pasan
- 1+ review
- Documentación actualizada
```

---

---

## Comandos diarios

### Backend

```bash
# Desarrollo
uvicorn app.main:app --reload

# Tests
pytest -v

# Migración
alembic revision --autogenerate -m "description"
alembic upgrade head

# Generar OpenAPI
curl http://localhost:8000/openapi.json > openapi.json
```

### Frontend

```bash
# Desarrollo
npm run dev

# Tests
npm run test

# Type check
npm run typecheck

# Build
npm run build

# Generar tipos
npx openapi-typescript http://localhost:8000/openapi.json -o src/api/generated/types.ts
```

---

---

## Checklist de inicio

- [ ] Crear repositorio GitHub (backend)
- [ ] Crear repositorio GitHub (frontend)
- [ ] Configurar PostgreSQL local
- [ ] Decisión de JWT/cookies ✅ httpOnly
- [ ] Decisión de state management ✅ Context + Zustand + TanStack
- [ ] Crear Proyectos GitHub (boards)
- [ ] Agregar documentación a ambos repos
- [ ] Primera reunión de sincronización (Frontend ↔ Backend)

---

## Contacto y sincronización

**Frecuencia mínima:** Diaria (15 min)

**Puntos clave:**
1. ¿Qué hice hoy?
2. ¿Qué bloqueos tengo?
3. ¿Qué necesita el otro equipo?
4. ¿Cambios en OpenAPI?

**Asincrónico:**
- GitHub Discussions
- Descripción de PR
- Mensajes en Discord/Slack

---

## Próximo paso

Elige por dónde empezar **mañana**:

1. **Backend Día 1:** Setup + openapi.json
2. **Frontend Día 1:** Setup + client + auth context
3. **Ambos en paralelo**

¿Tienes preguntas sobre algún día específico?

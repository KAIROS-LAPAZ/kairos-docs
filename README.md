# Kairos La Paz - Sistema Integral Digital

Bienvenido al repositorio oficial del Sistema Kairos La Paz.

## Que es Kairos?

Kairos es un movimiento catolico juvenil que busca formar lideres comprometidos con Cristo y la evangelizacion.

Este repositorio contiene el **Sistema Digital Kairos**: una plataforma web integral que centraliza toda la administracion de campamentos, inscripciones, pagos, tienda y comunicaciones de Kairos La Paz.

## Estructura del Proyecto

Este es un monorepo que contiene 3 modulos independientes:

### 1. **kairos-backend**

Backend API en FastAPI (Python). Contiene toda la logica de negocio, conexion a base de datos, autenticacion y APIs REST.

* **Tecnologia:** FastAPI, Python 3.13, PostgreSQL
* **Puerto:** 8000
* **Documentacion:** [kairos-backend/README.md](https://github.com/kairos-lapaz/kairos-backend)

### 2. **kairos-frontend**

Frontend web en React. Interfaz para usuarios, jovenes, coordinacion y tesoreria.

* **Tecnologia:** React 18, TypeScript, Axios
* **Puerto:** 3000
* **Documentacion:** [kairos-frontend/README.md](https://github.com/kairos-lapaz/kairos-frontend)

### 3. **kairos-docs**

Documentacion del proyecto. Especificaciones tecnicas, diagramas, guias, roadmap.

* **Contenido:** Docs tecnicas, ejecutivas, requisitos funcionales
* **Documentacion:** Este repositorio

## Vision del Proyecto

Transformar la administracion de Kairos de procesos manuales (WhatsApp, Excel, papel) a un sistema digital integrado donde:

* Los jovenes se inscriben en campamentos en linea
* Coordinacion gestiona campamentos, comisiones e inventarios
* Tesoreria ve pagos e ingresos en tiempo real
* La tienda Kairos genera ingresos para el grupo
* Todo esta organizado en una sola plataforma
* Ningun dato se pierde

## Roadmap de Desarrollo

| Fase           | Mes        |Hito                                             |
| -------------- | ------------ | ------------------------------------------------ |
| **v1.0 - MVP** | Sep-Oct 2026 | Pagina publica + Cuentas de usuario              |
|                | Oct-Nov 2026 | Campamentos + Inscripciones + Panel coordinacion |
|                |   | Panel tesoreria + Pagos                          |
|                | Nov-Dic 2026 | Tienda Kairos                                    |
|                | Dic-Ene 2026 | Pruebas, correcciones, lanzamiento oficial       |
| **v1.1**       | Ene-Feb 2026  | Notificaciones WhatsApp, exportaciones avanzadas |
| **v2.0**       | Feb+ 2026      | App movil, modo offline, chatbot con IA          |

## Quick Start para Desarrolladores

### Requisitos

* Python 3.13+
* Node.js 18+
* PostgreSQL 15+
* Git

### Setup Rapido

```bash
# 1. Clonar organizacion
git clone https://github.com/kairos-lapaz/kairos-backend.git
git clone https://github.com/kairos-lapaz/kairos-frontend.git

# 2. Backend
cd kairos-backend
python -m venv venv
venv\Scripts\activate  # Windows o source venv/bin/activate (Mac/Linux)
pip install -r requirements.txt
python app/main.py

# 3. Frontend (en otra terminal)
cd kairos-frontend
npm install
npm start

# Backend en: http://localhost:8000/docs
# Frontend en: http://localhost:3000
```

Ver documentacion completa en [kairos-backend/README.md](https://github.com/kairos-lapaz/kairos-backend) y [kairos-frontend/README.md](https://github.com/kairos-lapaz/kairos-frontend)

## Equipo

* **Samuel Alejandro Alvarez González** - Desarrollador Principal, Arquitecto
* **Lieras** - Frontend (si se suma)
* **Canedo** - Infraestructura (si se suma)
* **Tony** - Soporte (si se suma)

## Documentacion

* **[Documento Ejecutivo](./DOCUMENTO_EJECUTIVO.md)** - Para coordinadores, no tecnico
* **[Documento Tecnico](./DOCUMENTO_TECNICO.md)** - Para programadores
* **[Requisitos Funcionales](./REQUISITOS.md)** - Spec completa del sistema
* **[Base de Datos](./BASE_DE_DATOS.md)** - Diagrama ER, tablas, relaciones
* **[APIs](./APIS.md)** - Listado completo de endpoints

## Stack Tecnologico

| Capa              | Tecnologia            | Por Que                                           |
| ----------------- | --------------------- | ------------------------------------------------- |
| **Frontend**      | React + TypeScript    | Moderno, componentes reutilizables, tipado        |
| **Backend**       | FastAPI + Python      | Rapido, validacion automatica, APIs profesionales |
| **BD**            | PostgreSQL            | Relacional, robusto, escalable                    |
| **Autenticacion** | JWT (HS256)           | Seguro, stateless, estandar industria             |
| **ORM**           | SQLAlchemy            | Abstraccion segura de BD                          |
| **Server (Dev)**  | Uvicorn               | ASGI server para FastAPI                          |
| **Server (Prod)** | Gunicorn + Nginx      | Produccion robusta                                |
| **Seguridad**     | HTTPS + Let's Encrypt | Encriptacion automatica                           |

## Seguridad

* Contraseñas encriptadas (bcrypt)
* Autenticacion JWT
* Validacion de datos en servidor
* Permisos por rol
* HTTPS/SSL obligatorio
* Respaldos automaticos de BD
* No almacenamos datos de tarjetas (Mercado Pago)

## Caracteristicas Principales

### Para Jovenes

* Crear cuenta Kairos
* Ver informacion de Kairos
* Inscribirse a campamentos
* Comprar en tienda
* Ver mis inscripciones y pedidos
* Recibir notificaciones

### Para Coordinacion

* Crear campamentos
* Crear formularios dinamicos
* Ver inscritos en tiempo real
* Asignar usuarios a comisiones
* Gestionar inventarios por comision
* Crear tribus y asignar lideres
* Exportar listas a Excel

### Para Tesoreria

* Ver pagos de campamentos
* Registrar pagos (online, efectivo, beca)
* Ver ventas de tienda
* Registrar gastos (apostolados, materiales)
* Reportes financieros
* Graficas de ingresos/gastos

### Para Tienda

* Catalogo de productos
* Personalizacion (versiculos biblicos)
* Carrito de compras
* Pago online
* Recogida en sabados de comunidad
* Inventario automatico

## Como Contribuir

1. Crea una rama: `git checkout -b feature/mi-funcionalidad`
2. Haz cambios y prueba localmente
3. Commit: `git commit -m "Descripcion clara del cambio"`
4. Push: `git push origin feature/mi-funcionalidad`
5. Abre Pull Request en GitHub
6. Espera code review
7. Merge cuando este aprobado

## Notas Importantes

* **Nunca commitear** archivos `.env` (contienen contraseñas)
* **Usar `.env.example`** como plantilla
* **Validar datos en backend**, no confiar en frontend
* **Documentar codigo** con docstrings
* **Hacer commits pequeños** con mensajes descriptivos
* **Testear antes de pushear**

## FAQ

**P: Cual es la URL en produccion?**

R: https://kairoslapaz.com (cuando este lista)

**P: Cuanto cuesta mantener esto?**

R: ~$400 MXN/año (dominio) inicialmente. Despues $1500-2000 si usamos VPS.

**P: Que pasa si se cae el servidor?**

R: Con UPS aguanta ~2 horas sin luz. Respaldos automaticos cada noche.

**P: Como accedo a la BD?**

R: `psql -U postgres -d kairos_db` (solo Samuel y coordinador tecnico)

**P: Puedo usar esto para otro grupo?**

R: Si, el codigo es reutilizable. Contacta a Samuel para soporte.

## Soporte y Contacto

**Desarrollador:** Samuel Alejandro Alvarez
**Email:** [samuel@kairoslapaz.com](mailto:samuel@kairoslapaz.com)
**GitHub:** [Samuel's Profile](https://github.com/samuel-kairos-lapaz)
**WhatsApp:** (solo para urgencias)

---

## Licencia

Por definir.

---

**Ultima actualizacion:** 26 de Enero, 2025

**Version:** 1.0

**Estado:** En desarrollo - MVP fase 1

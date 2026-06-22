# Canasta

# **Video demonstration of front end: https://drive.google.com/file/d/14CMhteVgoQfbiSSDIiHfHjsdE-5SkH7Q/view?usp=sharing**

**Plataforma de comercio electrónico de abarrotes con recomendación de canasta de mercado (*market-basket*).**

Canasta es la interfaz productiva de un modelo de recomendación: por un lado ofrece a un cliente una experiencia de compra completa sobre el catálogo del socio formador, y por el otro entrega al área operativa un panel de inteligencia de negocio que consume el modelo en vivo.

> Reto · Tecnológico de Monterrey · Ingeniería en Ciencias de Datos y Matemáticas
> Equipo: **TCA Software Solutions**

---

## Tabla de contenido

- [Arquitectura](#arquitectura)
- [Requisitos](#requisitos)
- [Cómo ejecutar (Windows)](#cómo-ejecutar-windows)
- [Cuentas de demostración](#cuentas-de-demostración)
- [Estructura del proyecto](#estructura-del-proyecto)
- [API del backend](#api-del-backend)
- [Configuración](#configuración)
- [Funcionalidades](#funcionalidades)
- [Notas y limitaciones](#notas-y-limitaciones)

---

## Arquitectura

Arquitectura cliente–servidor de tres capas:

| Capa | Tecnología | Responsabilidad |
|------|------------|-----------------|
| Presentación | HTML + JavaScript (sin framework) | Experiencia de cliente y panel administrativo |
| Servicio | ASP.NET Core (.NET) | Compras (CSV), auditoría y proxy del recomendador |
| Modelo | FastAPI sobre Azure Container Apps | Recomendación de canasta de mercado |

- El **frontend** es un único archivo `index.html` con el catálogo embebido (7,738 productos), por lo que abre incluso sin backend para demostración.
- El **backend** sirve el frontend como contenido estático, persiste datos en archivos planos (JSON/CSV) y reenvía las consultas al modelo.
- El navegador **nunca** contacta directamente al modelo: las recomendaciones se enrutan por un proxy del backend para evitar problemas de CORS.

---

## Requisitos

> **El proyecto solo está soportado en Windows** (se ejecuta desde Visual Studio sobre el runtime de .NET).

- Windows 10 u 11
- Visual Studio 2022 o superior, con la carga de trabajo **ASP.NET y desarrollo web**
- SDK de .NET correspondiente a la versión del proyecto
- Conexión a internet (para el recomendador en la nube y el mapa en tiempo real)

---

## Cómo ejecutar (Windows)

El proyecto se entrega como un archivo comprimido (`.zip`).

1. **Descomprime** el `.zip` en una carpeta local (por ejemplo, en el Escritorio). Conserva la estructura de carpetas tal cual.
2. **Abre la solución** en Visual Studio con el archivo `.sln`.
3. **Revisa `appsettings.json`**: la carpeta de salida del CSV y la URL del recomendador deben corresponder a tu entorno.
4. **Restaura paquetes** (NuGet) y **compila** con `Ctrl+Shift+B`.
5. **Ejecuta** con `F5`. Visual Studio levanta el servidor local y abre el navegador (por defecto `https://localhost:7237`).
6. **Inicia sesión** con una de las cuentas de demostración.

Al correr, el frontend se sirve desde el propio backend (no necesitas otro servidor web). Las compras finalizadas se escriben en el CSV configurado, y el panel del colaborador muestra datos reales cuando ya existen registros.

---

## Cuentas de demostración

| Correo | Contraseña | Rol |
|--------|-----------|-----|
| `cristiano@gmail.com` | `cr7` | Usuario |
| `mbappe@orange.fr` | `paris10` | Usuario |
| `casillas@hotmail.com` | `saniker` | Usuario |
| `ramos@canarias.es` | `sergio4` | Usuario |
| `messi@tca.com` | `colab123` | Colaborador (panel admin) |

---

## Estructura del proyecto

```
RecepcionFacturasWeb/
├─ Controllers/
│  ├─ AuthController.cs            # Login por correo/contraseña y roles
│  ├─ ComprasController.cs         # POST/GET de compras (escribe CSV)
│  ├─ RecomendacionesController.cs # Proxy al modelo de Azure
│  ├─ AuditoriaController.cs       # Bitácora para el panel del colaborador
│  └─ AdminController.cs
├─ Services/
│  ├─ CompraService.cs            # Persistencia de compras (CSV + JSON)
│  ├─ UsuariosService.cs
│  └─ AuditoriaService.cs
├─ Models/
│  ├─ Compra.cs
│  ├─ Usuario.cs
│  └─ RegistroAuditoria.cs
├─ wwwroot/
│  ├─ index.html                  # Toda la aplicación (frontend)
│  └─ catalogo_front.json         # Catálogo (también embebido en el HTML)
├─ usuarios.json                  # Seed de usuarios (login)
├─ appsettings.json
└─ Program.cs
```

---

## API del backend

| Endpoint | Método | Descripción |
|----------|--------|-------------|
| `/api/auth/login` | POST | Autenticación `{ rfc, password }` |
| `/api/compras` | POST | Registra una compra y la escribe en el CSV |
| `/api/compras` | GET | Lista compras (todas para colaborador; propias para usuario) |
| `/api/recomendaciones` | POST | Proxy: envía `{ products: [...] }` al modelo y devuelve su ranking |
| `/api/auditoria` | GET | Bitácora de actividad (solo colaborador) |

La identificación se envía por encabezados HTTP: `X-User-RFC` (usuario) y `X-Admin-RFC` (rol colaborador).

---

## Configuración

En `appsettings.json`:

```json
{
  "Canasta": {
    "CarpetaCsv": "C:\\ruta\\donde\\se\\guarda\\compras.csv",
    "RecomendadorUrl": "https://reco-basket-app.gentlesea-cb2510e3.eastus.azurecontainerapps.io"
  }
}
```

- `CarpetaCsv`: carpeta donde el backend escribe `compras.csv`. Si se deja vacía, usa la carpeta `bin`.
- `RecomendadorUrl`: URL base del modelo de recomendación.

Recuerda registrar el servicio de compras en `Program.cs`:

```csharp
builder.Services.AddSingleton<CompraService>();
```

---

## Funcionalidades

**Cliente**
- Catálogo completo (7,738 productos) con búsqueda, filtro por categoría y paginación.
- Al elegir categoría: sección destacada ("Nuestra selección") y luego el resto.
- Carrito, envío gratis por monto, y pago en 3 pasos (entrega → pago → confirmación).
- Captura de tarjeta con formato automático y validación de CP de cobertura.
- Tiempo estimado de entrega y mensaje de preparación del pedido.
- "Mis compras" con estados y método de pago usado.
- "Mi cuenta" persistente (datos, direcciones y tarjetas se guardan en el navegador).

**Colaborador (panel)**
- KPIs de ventas y ganancia, ticket promedio, productos más vendidos.
- Ventas por categoría y por ciudad (con divisa local de referencia).
- Mapa de calor por día/hora, productos estrella, segmentos de clientes y predicción de próxima compra.
- **Recomendador** conectado al modelo en vivo (1 a 10 claves → ranking).
- **Catálogo** en 3 vistas (General / Por ciudad / Proveedores) estilo tabla empresarial.
- **Usuarios**: ficha de perfilado por cliente.
- **Zonas de entrega**: cobertura por diagrama de Voronoi; botón *Consultar Tiempo Real* que superpone el Voronoi sobre OpenStreetMap (Leaflet).
- **Auditoría** conectada al backend.

---

## Notas y limitaciones

- **Solo Windows** (ejecución desde Visual Studio / runtime .NET).
- El **mapa en tiempo real** y el **recomendador** requieren internet; sin conexión, la app usa el mapa estilizado y datos simulados de demostración.
- Los **precios base** están en MXN (lista del socio formador). Las divisas por ciudad y los factores de margen son **supuestos de demostración**, etiquetados como tales en la interfaz.
- La **autenticación es de prototipo**: identificación por encabezados y contraseñas en texto plano en `usuarios.json`. No usar tal cual en producción.
- La persistencia es sobre **archivos planos** (JSON/CSV); para producción se recomienda migrar a una base de datos.
- Si en Auditoría aparecen registros antiguos (facturas/SOFTPLUS), borra el archivo `auditoria.json` de la carpeta `bin` para limpiar la bitácora.

---

*Desarrollado por TCA Software Solutions.*

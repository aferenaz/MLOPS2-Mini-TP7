# Mini-TP 7 · Seguridad con SAIF — gRPC sobre el modelo de demanda de Gout

**Operaciones de Aprendizaje Automático II · CEIA – FIUBA** · Andrea Ferenaz

[![Abrir en Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/aferenaz/MLOPS2-Mini-TP7/blob/main/mini_tp7_grpc_gout.ipynb)

Asegura el servicio gRPC que expone el **pronóstico de demanda de Gout** (cadena de panaderías sin gluten:
9 sucursales y 96 SKUs). Es el mismo servicio gRPC del TP final, ahora con los controles de la capa gRPC de SAIF.

## Qué se implementó

| Entrega | Implementación |
|---|---|
| Control principal | Interceptor gRPC que verifica un **JWT** (firma HS256, vencimiento, emisor) y autoriza por **rol** |
| Sobre el modelo propio | El servicio carga el modelo de Gout (`modelo_gout.joblib`) y **valida la entrada** antes de predecir |
| Anti-DoS | Límite de **64 KB** por mensaje (una consulta legítima pesa menos de 100 bytes) |
| Canal | Paso de `insecure_port` a **TLS**, con demostración usando un certificado autofirmado; mTLS para servicios internos |
| Gobernanza | **Logging de auditoría** en JSON por llamada (sin features crudas, solo su hash) y **model card** con el SHA-256 del modelo |
| Reflexión | Amenaza mitigada por cada control y su elemento de SAIF (sección 4 del notebook) |

## Resultados de la prueba

| Caso | Respuesta del servicio |
|---|---|
| Sin token | `UNAUTHENTICATED` |
| Token vencido | `UNAUTHENTICATED` |
| Token firmado con otro secreto | `UNAUTHENTICATED` |
| Rol `invitado` | `PERMISSION_DENIED` |
| Features incompletas | `INVALID_ARGUMENT` |
| Mensaje de 200 KB | `RESOURCE_EXHAUSTED` (se descarta antes de deserializar) |
| Rol `planificador` | OK, devuelve unidades estimadas |
| Rol `sistema` (DAG de Airflow) | OK, devuelve unidades estimadas |
| Cliente sin cifrar contra puerto TLS | `UNAVAILABLE` |

## Controles y SAIF

| Control | Amenaza que mitiga | Elemento de SAIF |
|---|---|---|
| JWT + rol en el interceptor | Acceso no autorizado al pronóstico | Bases de seguridad sólidas |
| Mismo esquema JWT/rol en REST, GraphQL y gRPC | Políticas distintas según el punto de acceso | Armonizar controles de plataforma |
| TLS / mTLS | Lectura del pronóstico o robo del token en tránsito | Bases de seguridad sólidas |
| Límite de tamaño y validación de entrada | Denegación de servicio y entradas mal formadas | Bases de seguridad sólidas |
| Auditoría JSON | Intentos de acceso sin rastro | Extender detección y respuesta |
| Model card + SHA-256 | Reemplazo silencioso del modelo; uso fuera de alcance | Contextualizar riesgos en el proceso de negocio |

## Cómo correrlo

### En Colab

1. Abrir el notebook con el botón **Abrir en Colab** de arriba.
2. En la celda de instalación, descomentar la línea `%pip install ...` y correrla.
3. **Entorno de ejecución → Reiniciar sesión** (`grpcio-tools` actualiza `protobuf`).
4. Subir `modelo_gout.joblib` a `/content` (o definir `GOUT_MODELO_PATH` con su ruta en Drive).
5. **Entorno de ejecución → Ejecutar todas.**

### Local, con uv

```bash
uv add grpcio grpcio-tools pyjwt scikit-learn numpy pandas joblib cryptography xgboost
uv run python -m ipykernel install --user --name mlops2
uv run jupyter lab
```

No necesita Docker: el servidor gRPC corre dentro del mismo proceso del notebook.

### Variables de entorno

| Variable | Para qué | Por defecto |
|---|---|---|
| `GOUT_MODELO_PATH` | Ruta del modelo entrenado | `modelo_gout.joblib` |
| `GOUT_JWT_SECRET` | Secreto para firmar y verificar los JWT | Valor de demo (no usar en producción) |
| `GOUT_MODELO_SHA256` | Hash esperado del modelo; si no coincide, el servicio no arranca | Sin verificación |

## Modelo

El modelo de Gout **no se sube al repo** (está en `.gitignore`): es pesado y está entrenado con datos
comercialmente sensibles. El notebook acepta un estimador suelto (por ejemplo `XGBRegressor`) o un diccionario
`{"modelo", "features", "version"}` guardado con `joblib`. La lista `FEATURES` del notebook tiene que respetar el
orden con que se entrenó el modelo.

Si `modelo_gout.joblib` no está, el notebook entrena un **modelo de reemplazo** con datos sintéticos, solo para
poder correr de punta a punta.

## Archivos

| Archivo | Contenido |
|---|---|
| `mini_tp7_grpc_gout.ipynb` | Notebook de la entrega |
| `MODEL_CARD_gout.md` | Model card; la genera el notebook al correr con el modelo real |
| `requirements.txt` | Dependencias |
| `.gitignore` | Excluye modelo, auditoría, código generado, secretos y certificados |

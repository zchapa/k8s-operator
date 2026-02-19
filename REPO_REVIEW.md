# Review del Repositorio OpenClaw Kubernetes Operator

## 1. Análisis de Seguridad

Este repositorio implementa un operador de Kubernetes con un fuerte enfoque en la seguridad ("Secure by Default"). Las características de seguridad incluyen:

*   **Ejecución No-Root:** Todos los contenedores principales y sidecars (excepto Ollama) se ejecutan como usuario no-root (UID 1000/65532).
*   **Sistema de Archivos de Solo Lectura:** El sistema de archivos raíz es de solo lectura por defecto, con volúmenes específicos (`~/.openclaw/`, `/tmp`) para escritura.
*   **Capabilities Limitadas:** Se eliminan todas las capacidades de Linux (`ALL`) por defecto.
*   **Seccomp Profile:** Se habilita `RuntimeDefault` por defecto.
*   **Network Policies:** Se aplican políticas de red "Default Deny", permitiendo solo tráfico DNS y HTTPS de salida, e ingreso limitado al mismo namespace.
*   **Webhook de Validación:** Un webhook valida configuraciones inseguras, bloqueando la ejecución como root y advirtiendo sobre configuraciones riesgosas (falta de TLS en Ingress, privilegios elevados, etc.).
*   **Image Digest Pinning:** Se recomienda y advierte sobre el uso de digests inmutables para imágenes de contenedores.
*   **ServiceAccount Dedicada:** Cada instancia tiene su propia ServiceAccount con permisos mínimos (RBAC).
*   **Secrets:** Integración con Secrets de Kubernetes para API keys y tokens, evitando exponer credenciales en texto plano.

**Conclusión:** El repositorio es altamente seguro y sigue las mejores prácticas de seguridad en Kubernetes.

## 2. Lista de Características

El operador proporciona una gestión completa del ciclo de vida de los agentes OpenClaw:

*   **Despliegue Declarativo (CRD):** Un solo recurso `OpenClawInstance` define todo el stack (StatefulSet, Service, Ingress, RBAC, PVC, etc.).
*   **Gestión de Configuración:** Soporte para configuración inline (JSON/JSON5) o externa (ConfigMap), con modos de sobrescritura (`overwrite`) o fusión (`merge`) para preservar cambios en tiempo de ejecución.
*   **Sidecars Integrados:**
    *   **Chromium:** Para automatización de navegador.
    *   **Ollama:** Para inferencia de LLM local (requiere root, pero está aislado).
    *   **Tailscale:** Para acceso seguro a la red tailnet sin exponer puertos públicos.
*   **Init Containers:**
    *   **Runtime Deps:** Instalación automática de pnpm y Python/uv.
    *   **Skills:** Instalación declarativa de habilidades de ClawHub.
    *   **Workspace:** Sembrado inicial de archivos y directorios en el PVC.
*   **Persistencia:** Soporte para PVCs con backup y restauración integrados.
*   **Backup y Restauración:** Integración nativa con Backblaze B2 para snapshots y recuperación ante desastres.
*   **Auto-Actualización:** Polling de registro OCI, backups automáticos antes de actualizar, y rollback automático en caso de fallo.
*   **Observabilidad:** Métricas de Prometheus, integración con ServiceMonitor, y logs estructurados.
*   **Resiliencia:** PodDisruptionBudgets, Liveness/Readiness/Startup probes configurables.

## 3. Puntos Faltantes o Potenciales Fallos

A pesar de ser robusto, existen limitaciones y áreas de mejora:

*   **Escalado Horizontal (HPA):** Limitado a una sola réplica (`Replicas: 1`). No soporta escalado horizontal nativo, lo que limita la capacidad de carga.
*   **Alta Disponibilidad (HA):** Al ser un StatefulSet de una sola réplica con PVC local (generalmente atado a una zona), la caída de la zona implica tiempo de inactividad hasta que el volumen se pueda mover o restaurar.
*   **Almacenamiento de Backups:** Limitado exclusivamente a Backblaze B2. No hay soporte nativo para AWS S3, Google Cloud Storage o Azure Blob Storage.
*   **Base de Datos Externa:** Dependencia de SQLite o archivos locales en el PVC. No hay soporte nativo en el CRD para conectar bases de datos externas (PostgreSQL, etc.), aunque se pueden usar sidecars personalizados.
*   **Gestión de Secretos Avanzada:** Requiere la creación manual de Secrets de Kubernetes. No hay integración nativa con Vault, AWS Secrets Manager o similares (aunque se puede lograr con CSI drivers y `extraVolumeMounts`).
*   **Seguridad de Ollama:** El sidecar de Ollama requiere ejecutarse como root, lo cual debilita ligeramente el modelo de seguridad si se habilita.
*   **Compatibilidad JSON5/Merge:** No se puede usar el formato `json5` junto con el modo `merge`, lo cual es una limitación de configuración validada por el webhook.
*   **Límites de Recursos:** Aunque el webhook advierte, no impone límites de recursos por defecto si el usuario no los define, lo que podría llevar a problemas de "Noisy Neighbor" en el clúster.
*   **Ingress Avanzado:** El soporte de Ingress es básico. No soporta Gateway API ni funcionalidades avanzadas de tráfico (Canary, Blue/Green) de forma nativa.

# Declaración de uso de IA y Referencias

## Declaración de uso de Inteligencia Artificial

Se utilizó Claude (Anthropic) como apoyo en el desarrollo de esta Evaluación Final
Transversal, específicamente en:

- Diseño inicial de los workflows de GitHub Actions reutilizables
  (`build-push-template.yaml`, `deploy-bluegreen-template.yaml`) y de la
  estrategia Blue-Green sobre K3s.
- Diseño del mecanismo de auto-remediación (watchdog) y su configuración RBAC.
- Redacción inicial de la documentación técnica (`README-ARQUITECTURA.md`).
- Apoyo en la depuración de errores reales encontrados durante la implementación:
  un bug de indentación YAML en la generación dinámica de variables de entorno,
  un problema de compatibilidad de imagen de contenedor para el watchdog
  (bitnami/kubectl y rancher/kubectl no incluían shell; se resolvió usando
  alpine/k8s), y un problema de propagación de outputs entre jobs de GitHub
  Actions cuando el valor contenía un secret (se resolvió reconstruyendo la
  referencia de imagen dentro del job de deploy en vez de pasarla como output
  entre jobs).

Todo el código fue **implementado, ejecutado y verificado personalmente** contra
el clúster K3s real de la estudiante, incluyendo:

- Ejecución exitosa del pipeline completo (Build → Deploy Blue-Green → Health
  Gate → Promoción) en GitHub Actions.
- Verificación manual vía SSH y `kubectl` del estado del clúster tras cada
  despliegue.
- Simulación en vivo de un fallo de producción (imagen rota inyectada
  manualmente) y verificación de que el watchdog detectó el fallo y revirtió
  el tráfico automáticamente en aproximadamente 15 segundos, sin intervención
  humana (evidencia y logs completos en la sección 10 de
  `README-ARQUITECTURA.md`).

La estudiante comprende y puede explicar el funcionamiento de cada componente
del sistema (plantillas reutilizables, Service/Deployment Blue-Green, RBAC del
watchdog, lógica de Health Gate y rollback).

## Referencias (formato APA 7)

Amazon Web Services. (s. f.). *Building a Kubernetes cluster with EKS*. AWS
Documentation. https://docs.aws.amazon.com/eks/

Kubernetes. (s. f.). *Services*. Kubernetes Documentation.
https://kubernetes.io/docs/concepts/services-networking/service/

GitHub. (s. f.). *Reusing workflows*. GitHub Docs.
https://docs.github.com/actions/using-workflows/reusing-workflows

Rancher (SUSE). (s. f.). *K3s: Lightweight Kubernetes*. https://k3s.io/

# Declaración de uso de IA y Referencias

## Declaración de uso de Inteligencia Artificial

> Completar con datos reales antes de entregar.

Se utilizó la herramienta de IA **[nombre de la herramienta, ej. Claude/ChatGPT]** como
apoyo en las siguientes tareas del desarrollo de esta Evaluación Final Transversal:

- [ ] Diseño y redacción inicial de los workflows de GitHub Actions (`build-push-template.yaml`,
      `deploy-bluegreen-template.yaml`).
- [ ] Diseño del mecanismo de auto-remediación (`watchdog`) y su RBAC.
- [ ] Redacción de la documentación técnica (`README-ARQUITECTURA.md`).
- [ ] Depuración de errores durante las pruebas (detallar cuáles).

Todo el código generado fue **revisado, probado y adaptado manualmente** por el/la
estudiante contra el clúster K3s real antes de la entrega, verificando que:

- Los pipelines se ejecutan correctamente en GitHub Actions.
- El Health Gate y la promoción/rollback funcionan según lo documentado.
- El watchdog detecta y remedia fallos inyectados manualmente (ver sección 9 de
  `README-ARQUITECTURA.md`).

Prompt(s) principal(es) utilizado(s) (resumen): *[completar]*

## Referencias (formato APA 7)

> Completar/ajustar según las fuentes efectivamente consultadas.

Amazon Web Services. (s. f.). *Building a Kubernetes cluster with EKS*. AWS Documentation.
https://docs.aws.amazon.com/eks/

Kubernetes. (s. f.). *Services*. Kubernetes Documentation. https://kubernetes.io/docs/concepts/services-networking/service/

GitHub. (s. f.). *Reusing workflows*. GitHub Docs. https://docs.github.com/actions/using-workflows/reusing-workflows

Rancher (SUSE). (s. f.). *K3s: Lightweight Kubernetes*. https://k3s.io/

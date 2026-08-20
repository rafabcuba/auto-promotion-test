# Migración a esquema de Label-Based Promotion (dev → master)

## 0. Objetivo del documento

Definir el procedimiento para migrar el repositorio al esquema **Label-Based Promotion**, partiendo de un `dev` actual que tiene discrepancias no auditadas contra `master`. La estrategia es hacer un **corte limpio y organizado**: se conserva el `dev` actual como archivo histórico, y se crea un `dev` nuevo sincronizado desde `master`, que será la base sobre la cual se aplica el nuevo flujo de trabajo hacia adelante.

---

## 1. Plan de migración de ramas

### 1.1 Congelar y renombrar el `dev` actual

No se borra nada. El `dev` actual pasa a ser un archivo de referencia histórica.

```bash
git fetch origin

# Renombrar dev actual a dev-legacy (se conserva íntegro como respaldo/consulta)
git checkout dev
git checkout -b dev-legacy
git push origin dev-legacy

# Documentar el punto exacto de corte
git tag corte-migracion-$(date +%Y%m%d) origin/master
git push origin corte-migracion-$(date +%Y%m%d)
```

> `dev-legacy` queda disponible para hacer `git log` o `cherry-pick` puntual de algo que se haya quedado atrás y se detecte más adelante. No se toca ni se borra hasta que el equipo confirme que ya no se necesita (sugerido: mantener mínimo 2-3 meses).

### 1.2 Crear el nuevo `dev` desde `master`

```bash
git checkout master
git pull origin master
git checkout -b dev
git push origin dev
```

A partir de aquí, `dev` = `master` exactamente. Cero discrepancia. Punto de partida limpio.

### 1.3 Eliminar la rama `dev` antigua del remoto y liberar el nombre

```bash
git push origin --delete dev   # solo si "dev-legacy" ya se hizo push exitosamente
```

> ⚠️ Coordinar con el equipo el momento exacto de este corte (idealmente un día de baja actividad, ej. viernes fin de día o inicio de sprint). Cualquier rama `feature/*` local basada en el `dev` viejo deberá rebasear contra el `dev` nuevo.

### 1.4 Reconciliar el backlog pendiente de `dev-legacy`

Antes de que el equipo empiece a trabajar sobre el `dev` nuevo, se audita qué había en `dev-legacy` que aún es válido y no llegó a `master`:

```bash
git log origin/master..origin/dev-legacy --oneline --no-merges
```

Cada commit/PR de esa lista se reabre como un **PR nuevo, limpio**, contra el `dev` recién creado (no se hace cherry-pick directo del legacy a master — se reintroduce por el flujo normal para que pase por CI y quede trazado desde el día uno del nuevo esquema).

### 1.5 Actualizar la protección de ramas en GitHub

- `master`: PR obligatorio, checks requeridos (CI completo), 1-2 aprobaciones, prohibir push directo y force-push.
- `dev` (nuevo): PR obligatorio, checks básicos (lint + tests unitarios), 1 aprobación.
- Actualizar el workflow de CI si estaba referenciando la rama `dev` por nombre en algún filtro.

### 1.6 Comunicación al equipo

Enviar un aviso formal (Slack/correo) con:
- Fecha y hora exacta del corte.
- Instrucción de qué hacer con ramas locales (`git fetch && git rebase origin/dev` contra el nuevo `dev`).
- Aclarar que `dev-legacy` queda solo como consulta, no se debe seguir trabajando ahí.

---

## 2. Buenas prácticas para PRs hacia `dev`

Estas reglas aplican desde el día uno del `dev` nuevo, para evitar que se repita el desorden que motivó la migración.

### 2.1 Tamaño y alcance
- Un PR = un cambio lógico y acotado (una funcionalidad, un fix, un módulo). Evitar PRs que mezclen features no relacionados — dificulta saber qué promocionar y qué no.
- Preferir PRs pequeños y frecuentes sobre PRs gigantes esporádicos. Facilita revisión, CI más rápido y promoción más granular.

### 2.2 Nomenclatura de ramas origen
```
feature/<módulo>-<descripción-corta>
fix/<módulo>-<descripción-corta>
hotfix/<módulo>-<descripción-corta>   # solo para directo a master, ver sección 4.4
```

### 2.3 Checklist antes de abrir el PR
- [ ] Rebase contra `dev` actualizado (no merge, para mantener historia lineal legible).
- [ ] CI local ejecutado (lint + tests) antes de subir, no depender solo del pipeline remoto.
- [ ] Descripción del PR explica **qué cambia y por qué**, no solo "fix bug".
- [ ] Vincular el PR al issue/ticket correspondiente si aplica.
- [ ] Sin cambios de configuración/credenciales hardcodeadas.

### 2.4 En el merge a `dev`
- Usar **Squash and merge** para PRs de feature/fix pequeños → deja un commit limpio y atómico en `dev`, lo cual es clave porque ese commit único es exactamente lo que luego se cherry-pickea a `master`.
- Usar **Merge commit** solo si el PR realmente necesita preservar sub-historia (raro, evitar por defecto).
- El título del commit de squash debe ser descriptivo — será el mismo título que aparecerá en el PR de promoción a `master`.

### 2.5 Etiquetado de estado en `dev`
- `in-review`: recién abierto.
- `needs-changes`: observaciones pendientes.
- `merged-dev`: ya en `dev`, pendiente de validación adicional (QA/staging).
- `ready-for-master`: dispara la automatización de promoción (ver sección 3).
- `blocked` / `wip`: explícitamente no listo, para que quede visible que no debe promoverse aunque esté mergeado en `dev`.

---

## 3. Automatización de la promoción (recordatorio del mecanismo)

```yaml
name: Promote to master
on:
  pull_request:
    types: [labeled]
permissions:
  contents: write
  pull-requests: write
  issues: write
jobs:
  promote:
    if: github.event.label.name == 'ready-for-master'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - name: Cherry-pick to master
        id: cherry
        continue-on-error: true
        run: |
          git config user.name "github-actions"
          git config user.email "actions@github.com"
          git checkout master
          git checkout -b promote/pr-${{ github.event.pull_request.number }}
          git cherry-pick -x -m 1 ${{ github.event.pull_request.merge_commit_sha }}
          git push origin promote/pr-${{ github.event.pull_request.number }}
      - name: Create PR
        if: steps.cherry.outcome == 'success'
        run: |
          gh pr create --base master --head promote/pr-${{ github.event.pull_request.number }} \
            --title "Promote: ${{ github.event.pull_request.title }}" \
            --body "Auto-promovido desde dev PR #${{ github.event.pull_request.number }}"
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      - name: Notify conflict
        if: steps.cherry.outcome == 'failure'
        run: |
          gh pr comment ${{ github.event.pull_request.number }} \
            --body "⚠️ Conflicto al promover automáticamente a master. Requiere resolución manual (ver sección 4 del documento de proceso)."
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

---

## 4. Resolución de conflictos en la promoción

### 4.1 Por qué ocurren
Un cherry-pick de `dev` a `master` falla típicamente cuando:
- `master` recibió un **hotfix** directo que `dev` no tiene todavía.
- Dos PRs promovidos en distinto orden tocan las mismas líneas.
- `dev` tiene cambios posteriores (de otro PR ya mergeado) sobre el mismo archivo que alteran el contexto necesario para aplicar el patch limpio.

### 4.2 Procedimiento manual cuando falla el cherry-pick automático

```bash
git fetch origin
git checkout master
git checkout -b promote/pr-<numero>
git cherry-pick -x -m 1 <merge-commit-sha>

# Si hay conflicto:
git status                      # ver archivos en conflicto
# resolver manualmente cada archivo
git add <archivo-resuelto>
git cherry-pick --continue

git push origin promote/pr-<numero>
gh pr create --base master --head promote/pr-<numero> \
  --title "Promote (resuelto manualmente): <título original>" \
  --body "Conflicto resuelto manualmente. Ver detalle en comentarios del PR original #<numero>."
```

### 4.3 Responsable de resolver conflictos
Debe resolverlo **el autor original del PR** (o el aprobador técnico del módulo vía CODEOWNERS), no una persona ajena al cambio — es quien mejor conoce la intención del código y puede decidir correctamente cuál versión debe prevalecer.

### 4.4 Caso especial: hotfixes directos a `master`
Si `master` necesita un fix urgente que no puede esperar el ciclo normal:
1. Rama `hotfix/*` creada desde `master`.
2. PR directo a `master` (pasa por el mismo branch protection).
3. **Inmediatamente después del merge**, se hace el proceso inverso: cherry-pick de ese hotfix hacia `dev`, para que no se pierda ni genere conflictos futuros en la siguiente promoción.

```bash
git checkout dev
git cherry-pick -x <hotfix-commit-sha>
git push origin dev
```

Esto es crítico: **todo cambio directo a `master` debe reflejarse en `dev` el mismo día**, o se acumula deuda de sincronización otra vez — exactamente el problema que se resolvió con la migración.

### 4.5 Prevención (reduce conflictos antes de que ocurran)
- PRs pequeños y frecuentes (sección 2.1) reducen la probabilidad de solapamiento.
- Evitar dejar PRs "colgados" mucho tiempo sin promover — mientras más tiempo pasa, más diverge `dev` respecto al estado de `master` en el que se originó el cambio.

---

## 5. ¿Es importante el orden cronológico de promoción a `master`?

**Sí, y es uno de los puntos más delicados del esquema.** No es obligatorio en el sentido estricto de "PR más viejo primero", pero **sí importa el orden de dependencia**, y ambos conceptos suelen confundirse.

### 5.1 Qué es lo que realmente importa: orden de dependencia, no de antigüedad

Si el PR B fue construido sobre cambios introducidos por el PR A (aunque A todavía no esté en `master`), **A debe promoverse antes que B**, sin importar cuál se abrió primero cronológicamente.

Antes de etiquetar `ready-for-master`, preguntarse:
- ¿Este cambio depende de otro archivo/función que fue modificado por un PR que aún no está en `master`?
- Si se promueve este PR solo, ¿`master` queda en un estado consistente y funcional?

### 5.2 Riesgo de ignorar el orden de dependencia
Si promocionas B sin A:
- El cherry-pick de B puede fallar (referencia código que no existe en `master` todavía) — esto de hecho actúa como una salvaguarda automática, el conflicto te avisa.
- Peor caso: el cherry-pick **aplica sin conflicto pero deja `master` en estado roto** (ej. B llama a una función que A define, pero el patch aplica igual porque el conflicto no es a nivel de líneas sino de lógica). Esto **no lo detecta git, lo detecta el CI** — de ahí que sea innegociable correr el pipeline completo también en el PR de promoción hacia `master`, no confiar solo en que ya pasó en `dev`.

### 5.3 Qué hacer en la práctica
- Al etiquetar PRs como `ready-for-master`, revisar si hay dependencias entre ellos y etiquetarlos/promoverlos en el orden correcto (usar el Project Board de la sección 6 para visualizar esto).
- Si dos PRs `ready-for-master` son completamente independientes entre sí, el orden cronológico entre ellos **no importa** — pueden promoverse en cualquier orden o incluso en paralelo.
- El pipeline de CI en el PR `dev → master` es el último seguro: si algo quedó inconsistente por orden incorrecto, debe fallar ahí antes de mergear, nunca detectarse ya en producción.

---

## 6. Trazabilidad y visibilidad (recomendado, no obligatorio)

GitHub Project (board) con columnas:

```
En dev → Validando (QA/staging) → Ready for master → Promovido a master
```

Cada PR se mueve manualmente o vía automation rules de GitHub Projects cuando cambia de label. Esto da visibilidad de estado sin depender de memoria del equipo, y hace evidente de un vistazo si hay dependencias pendientes antes de promover algo.

---

## 7. Checklist resumen de la migración

- [ ] Renombrar `dev` actual → `dev-legacy`, taguear el punto de corte.
- [ ] Crear `dev` nuevo desde `master` (sincronizado 100%).
- [ ] Auditar `dev-legacy` y reabrir como PRs nuevos lo que siga siendo válido.
- [ ] Configurar branch protection en `dev` y `master`.
- [ ] Instalar workflow de auto-promoción por label.
- [ ] Instalar/actualizar CODEOWNERS si aplica.
- [ ] Comunicar al equipo la fecha de corte y el nuevo flujo.
- [ ] Documentar buenas prácticas de PR (sección 2) en el `CONTRIBUTING.md` del repo.
- [ ] Definir responsable de resolver conflictos de promoción (sección 4.3).
- [ ] Establecer GitHub Project board para trazabilidad (sección 6).

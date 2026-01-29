# Plan de Mejoras del Frontend - AcaInfo

## Resumen Ejecutivo

Analisis exhaustivo del frontend de AcaInfo (React 19 + TypeScript + TanStack Query + Tailwind CSS) identificando mejoras de UX, seguridad y calidad visual. Las mejoras estan priorizadas de **mas simple a mas compleja** y de **mas critica a menos critica**, con enfoque en cambios frontend.

**Principio clave:** Los usuarios no administradores deben ver la menor cantidad de datos administrativos posible (IDs, codigos internos, etc.).

---

## PRIORIDAD ALTA - Criticas + Simples

### 1. Eliminar Header X-User-Id del apiClient
**Archivo:** `acainfo-front/src/shared/services/apiClient.ts` (lineas 18-29)

**Problema:** Se envia el ID del usuario en cada request HTTP como header personalizado, exponiendo datos innecesariamente. Visible en Network tab del navegador.

**Codigo actual:**
```typescript
const authStorage = localStorage.getItem('auth-storage')
if (authStorage) {
  try {
    const { state } = JSON.parse(authStorage)
    if (state?.user?.id) {
      requestConfig.headers['X-User-Id'] = state.user.id.toString()
    }
  } catch (error) {
    console.error('Failed to parse auth storage:', error)
  }
}
```

**Solucion:** Eliminar este bloque completo. El backend debe obtener el usuario del token JWT.

**Complejidad:** Baja | **Impacto:** Seguridad

---

### 2. Ocultar IDs Numericos en Tablas Admin
**Archivos:**
- `acainfo-front/src/features/admin/users/components/UserTable.tsx` (linea 54)
- `acainfo-front/src/features/admin/payments/components/PaymentTable.tsx` (linea 79)
- `acainfo-front/src/features/admin/enrollments/components/EnrollmentTable.tsx`

**Problema:** Muestran `ID: {user.id}`, `ID: {payment.studentId}` que son datos internos sin valor para el usuario.

**Codigo actual (UserTable.tsx):**
```tsx
<div className="text-sm text-gray-500">ID: {user.id}</div>
```

**Solucion:** Eliminar las lineas que muestran IDs o reemplazar con identificadores naturales (email).

**Complejidad:** Baja | **Impacto:** Seguridad/UX

---

### 3. Ocultar Codigo de Asignatura en Vista Estudiante
**Archivos:**
- `acainfo-front/src/features/enrollments/components/EnrollmentDetailCard.tsx` (linea 26)
- `acainfo-front/src/features/payments/components/PaymentCard.tsx`

**Problema:** El `subjectCode` es un identificador interno (ej: "MAT-101") que no aporta valor al estudiante.

**Codigo actual:**
```tsx
<h1 className="text-2xl font-bold text-gray-900">{enrollment.subjectName}</h1>
<p className="mt-1 text-gray-500">{enrollment.subjectCode}</p>
```

**Solucion:** Eliminar la linea del codigo de asignatura en vistas de estudiante.

**Complejidad:** Baja | **Impacto:** UX

---

### 4. Agregar aria-labels a Botones de Icono
**Archivos prioritarios:**
- `acainfo-front/src/shared/components/layout/Sidebar.tsx` (boton cerrar linea 220)
- `acainfo-front/src/shared/components/layout/Header.tsx`

**Problema:** 0 atributos `aria-label` en todo el codebase. Los botones con iconos no son accesibles para lectores de pantalla.

**Codigo actual:**
```tsx
<button type="button" onClick={onClose} className="...">
  <svg>...</svg>
</button>
```

**Solucion:**
```tsx
<button
  type="button"
  onClick={onClose}
  className="..."
  aria-label="Cerrar menu de navegacion"
>
  <svg aria-hidden="true">...</svg>
</button>
```

**Complejidad:** Baja | **Impacto:** Accesibilidad

---

### 5. Reemplazar window.confirm() con Modal de Confirmacion
**Archivos afectados:** 17 archivos usan `window.confirm()`
- `acainfo-front/src/features/enrollments/pages/EnrollmentDetailPage.tsx`
- `acainfo-front/src/features/admin/payments/pages/AdminPaymentsPage.tsx`
- `acainfo-front/src/features/admin/payments/pages/AdminPaymentDetailPage.tsx`
- `acainfo-front/src/features/admin/groups/pages/AdminGroupsPage.tsx`
- `acainfo-front/src/features/admin/groups/pages/AdminGroupDetailPage.tsx`
- `acainfo-front/src/features/admin/enrollments/pages/AdminEnrollmentsPage.tsx`
- `acainfo-front/src/features/admin/enrollments/pages/AdminEnrollmentDetailPage.tsx`
- `acainfo-front/src/features/admin/teachers/pages/AdminTeachersPage.tsx`
- `acainfo-front/src/features/admin/teachers/pages/AdminTeacherDetailPage.tsx`
- `acainfo-front/src/features/admin/subjects/pages/AdminSubjectsPage.tsx`
- `acainfo-front/src/features/admin/subjects/pages/AdminSubjectDetailPage.tsx`
- `acainfo-front/src/features/admin/sessions/pages/AdminSessionGeneratePage.tsx`
- `acainfo-front/src/features/admin/sessions/pages/AdminSessionDetailPage.tsx`
- `acainfo-front/src/features/admin/schedules/pages/AdminGroupSchedulesPage.tsx`
- `acainfo-front/src/features/group-requests/pages/AdminGroupRequestsPage.tsx`
- `acainfo-front/src/features/group-requests/pages/AdminGroupRequestDetailPage.tsx`
- `acainfo-front/src/features/admin/payments/pages/AdminPaymentGeneratePage.tsx`

**Problema:** Los dialogs nativos del navegador son poco profesionales y no personalizables.

**Solucion:**
1. Crear componente `ConfirmDialog.tsx`:
```tsx
// acainfo-front/src/shared/components/common/ConfirmDialog.tsx
interface ConfirmDialogProps {
  isOpen: boolean
  title: string
  message: string
  confirmLabel?: string
  cancelLabel?: string
  variant?: 'danger' | 'warning' | 'info'
  onConfirm: () => void
  onCancel: () => void
}

export function ConfirmDialog({
  isOpen, title, message, confirmLabel = 'Confirmar',
  cancelLabel = 'Cancelar', variant = 'danger', onConfirm, onCancel
}: ConfirmDialogProps) {
  if (!isOpen) return null

  return (
    <div className="fixed inset-0 z-50 flex items-center justify-center">
      <div className="fixed inset-0 bg-black/50" onClick={onCancel} />
      <div className="relative bg-white rounded-lg p-6 max-w-md w-full mx-4 shadow-xl">
        <h3 className="text-lg font-semibold text-gray-900">{title}</h3>
        <p className="mt-2 text-gray-600">{message}</p>
        <div className="mt-6 flex justify-end gap-3">
          <button
            onClick={onCancel}
            className="px-4 py-2 text-sm font-medium text-gray-700 bg-white border border-gray-300 rounded-md hover:bg-gray-50"
          >
            {cancelLabel}
          </button>
          <button
            onClick={onConfirm}
            className={cn(
              "px-4 py-2 text-sm font-medium text-white rounded-md",
              variant === 'danger' && "bg-red-600 hover:bg-red-700",
              variant === 'warning' && "bg-yellow-600 hover:bg-yellow-700",
              variant === 'info' && "bg-blue-600 hover:bg-blue-700"
            )}
          >
            {confirmLabel}
          </button>
        </div>
      </div>
    </div>
  )
}
```

2. Crear hook `useConfirmDialog.ts`:
```tsx
// acainfo-front/src/shared/hooks/useConfirmDialog.ts
export function useConfirmDialog() {
  const [state, setState] = useState<{
    isOpen: boolean
    title: string
    message: string
    onConfirm: () => void
  }>({ isOpen: false, title: '', message: '', onConfirm: () => {} })

  const confirm = (title: string, message: string): Promise<boolean> => {
    return new Promise((resolve) => {
      setState({
        isOpen: true,
        title,
        message,
        onConfirm: () => {
          setState(s => ({ ...s, isOpen: false }))
          resolve(true)
        }
      })
    })
  }

  const cancel = () => {
    setState(s => ({ ...s, isOpen: false }))
  }

  return { state, confirm, cancel }
}
```

3. Migrar los 17 archivos

**Complejidad:** Media | **Impacto:** UX

---

### 6. Crear Componente LoadingState Unificado
**Patron duplicado en 20+ archivos:**
```tsx
<div className="flex h-64 items-center justify-center">
  <div className="h-8 w-8 animate-spin rounded-full border-4 border-blue-200 border-t-blue-600" />
</div>
```

**Solucion:**
```tsx
// acainfo-front/src/shared/components/ui/Spinner.tsx
interface SpinnerProps {
  size?: 'sm' | 'md' | 'lg'
  className?: string
}

const sizeClasses = {
  sm: 'h-4 w-4 border-2',
  md: 'h-8 w-8 border-4',
  lg: 'h-12 w-12 border-4',
}

export function Spinner({ size = 'md', className }: SpinnerProps) {
  return (
    <div
      className={cn(
        'animate-spin rounded-full border-blue-200 border-t-blue-600',
        sizeClasses[size],
        className
      )}
    />
  )
}

// acainfo-front/src/shared/components/common/LoadingState.tsx
export function LoadingState({ message }: { message?: string }) {
  return (
    <div className="flex h-64 flex-col items-center justify-center gap-3">
      <Spinner size="md" />
      {message && <p className="text-sm text-gray-500">{message}</p>}
    </div>
  )
}
```

**Complejidad:** Baja-Media | **Impacto:** Mantenibilidad

---

## PRIORIDAD MEDIA - Importantes pero mas esfuerzo

### 7. Crear Componente Badge Unificado
**18 componentes Badge con codigo duplicado:**
- `GroupStatusBadge.tsx`, `GroupTypeBadge.tsx`
- `PaymentStatusBadge.tsx`, `PaymentTypeBadge.tsx`
- `SessionModeBadge.tsx`, `SessionStatusBadge.tsx`, `SessionTypeBadge.tsx`
- `EnrollmentStatusBadge.tsx`
- `UserStatusBadge.tsx`, `RoleBadge.tsx`
- `SubjectStatusBadge.tsx`, `DegreeBadge.tsx`
- `ClassroomBadge.tsx`
- `GroupRequestStatusBadge.tsx`
- (mas duplicados entre admin y features)

**Solucion:**
```tsx
// acainfo-front/src/shared/components/ui/Badge.tsx
import { cva, type VariantProps } from 'class-variance-authority'

const badgeVariants = cva(
  'inline-flex items-center rounded-full px-2.5 py-0.5 text-xs font-medium',
  {
    variants: {
      variant: {
        default: 'bg-gray-100 text-gray-800',
        success: 'bg-green-100 text-green-800',
        warning: 'bg-yellow-100 text-yellow-800',
        error: 'bg-red-100 text-red-800',
        info: 'bg-blue-100 text-blue-800',
        purple: 'bg-purple-100 text-purple-800',
        orange: 'bg-orange-100 text-orange-800',
      },
    },
    defaultVariants: {
      variant: 'default',
    },
  }
)

interface BadgeProps extends VariantProps<typeof badgeVariants> {
  children: React.ReactNode
  className?: string
}

export function Badge({ variant, children, className }: BadgeProps) {
  return (
    <span className={cn(badgeVariants({ variant }), className)}>
      {children}
    </span>
  )
}

// acainfo-front/src/shared/config/badgeConfig.ts
export const PAYMENT_STATUS_CONFIG = {
  PENDING: { label: 'Pendiente', variant: 'warning' },
  PAID: { label: 'Pagado', variant: 'success' },
  CANCELLED: { label: 'Cancelado', variant: 'error' },
  OVERDUE: { label: 'Vencido', variant: 'error' },
} as const

export const USER_STATUS_CONFIG = {
  ACTIVE: { label: 'Activo', variant: 'success' },
  INACTIVE: { label: 'Inactivo', variant: 'default' },
  BLOCKED: { label: 'Bloqueado', variant: 'error' },
  PENDING_ACTIVATION: { label: 'Pendiente', variant: 'warning' },
} as const
```

**Complejidad:** Media | **Impacto:** Mantenibilidad/DRY

---

### 8. Extraer Iconos SVG a Lucide React
**Archivo:** `acainfo-front/src/shared/components/layout/Sidebar.tsx` (200+ lineas de SVG inline)

**Problema:** SVGs inline dificultan mantenimiento y aumentan tamano del bundle.

**Solucion:** Lucide React ya esta instalado en package.json:
```tsx
// ANTES
icon: (
  <svg className="h-5 w-5" fill="none" viewBox="0 0 24 24" strokeWidth="1.5" stroke="currentColor">
    <path strokeLinecap="round" strokeLinejoin="round" d="M2.25 12l8.954-8.955..." />
  </svg>
),

// DESPUES
import {
  Home, GraduationCap, Calendar, CreditCard,
  FileText, Book, Settings, Users, UserCircle,
  CheckCircle, Layers, Clock, ClipboardList
} from 'lucide-react'

icon: <Home className="h-5 w-5" />,
```

**Mapeo de iconos:**
| Seccion | Icono Lucide |
|---------|--------------|
| Dashboard | `Home` |
| Mis Inscripciones | `GraduationCap` |
| Sesiones | `Calendar` |
| Pagos | `CreditCard` |
| Materiales | `FileText` |
| Asignaturas | `Book` |
| Administracion | `Settings` |
| Usuarios | `Users` |
| Profesores | `UserCircle` |
| Inscripciones | `CheckCircle` |
| Grupos | `Layers` |
| Horarios | `Calendar` |
| Sesiones (admin) | `Clock` |
| Solicitudes | `ClipboardList` |

**Complejidad:** Media | **Impacto:** Mantenibilidad (-150 lineas de codigo)

---

### 9. Implementar Debounce en Filtros de Busqueda
**Archivo:** `acainfo-front/src/features/admin/payments/pages/AdminPaymentsPage.tsx`

**Problema:** Los filtros disparan requests en cada keystroke, generando carga innecesaria.

**Solucion:**
```tsx
// acainfo-front/src/shared/hooks/useDebounce.ts
import { useState, useEffect } from 'react'

export function useDebounce<T>(value: T, delay: number = 300): T {
  const [debouncedValue, setDebouncedValue] = useState<T>(value)

  useEffect(() => {
    const timer = setTimeout(() => setDebouncedValue(value), delay)
    return () => clearTimeout(timer)
  }, [value, delay])

  return debouncedValue
}

// Uso en AdminPaymentsPage
const [searchStudentId, setSearchStudentId] = useState('')
const debouncedStudentId = useDebounce(searchStudentId, 300)

useEffect(() => {
  const id = debouncedStudentId ? parseInt(debouncedStudentId, 10) : undefined
  setFilters((prev) => ({ ...prev, studentId: id, page: 0 }))
}, [debouncedStudentId])
```

**Complejidad:** Baja-Media | **Impacto:** Performance

---

### 10. Mejorar Mensajes de Error con Tipos Especificos
**Problema:** Todos los errores muestran el mismo mensaje generico.

**Solucion:**
```tsx
// acainfo-front/src/shared/components/common/ErrorState.tsx
interface ErrorStateProps {
  error: Error | unknown
  onRetry?: () => void
  title?: string
}

function getErrorMessage(error: unknown): string {
  if (error instanceof Error) {
    if (error.message.includes('Network Error')) {
      return 'Error de conexion. Verifica tu internet.'
    }
    if (error.message.includes('401') || error.message.includes('Unauthorized')) {
      return 'Tu sesion ha expirado. Por favor, inicia sesion nuevamente.'
    }
    if (error.message.includes('500')) {
      return 'Error del servidor. Intenta mas tarde.'
    }
    if (error.message.includes('400')) {
      return 'Datos invalidos. Revisa la informacion ingresada.'
    }
    if (error.message.includes('403')) {
      return 'No tienes permisos para realizar esta accion.'
    }
    if (error.message.includes('404')) {
      return 'El recurso solicitado no existe.'
    }
  }
  return 'Ocurrio un error inesperado. Intenta de nuevo.'
}

export function ErrorState({ error, onRetry, title = 'Error' }: ErrorStateProps) {
  return (
    <div className="rounded-lg border border-red-200 bg-red-50 p-6">
      <div className="flex items-start gap-3">
        <ExclamationCircle className="h-5 w-5 text-red-600 flex-shrink-0 mt-0.5" />
        <div>
          <h3 className="text-sm font-medium text-red-800">{title}</h3>
          <p className="mt-1 text-sm text-red-700">{getErrorMessage(error)}</p>
          {onRetry && (
            <button
              onClick={onRetry}
              className="mt-3 text-sm font-medium text-red-600 hover:text-red-800"
            >
              Reintentar
            </button>
          )}
        </div>
      </div>
    </div>
  )
}
```

**Complejidad:** Media | **Impacto:** UX

---

### 11. Agregar Componente Breadcrumbs
**Problema:** Las paginas de detalle solo tienen enlace "Volver a..." manual.

**Solucion:**
```tsx
// acainfo-front/src/shared/components/ui/Breadcrumbs.tsx
import { Link } from 'react-router-dom'
import { ChevronRight, Home } from 'lucide-react'

interface BreadcrumbItem {
  label: string
  href?: string
}

interface BreadcrumbsProps {
  items: BreadcrumbItem[]
}

export function Breadcrumbs({ items }: BreadcrumbsProps) {
  return (
    <nav aria-label="Breadcrumb" className="mb-4">
      <ol className="flex items-center gap-2 text-sm">
        {items.map((item, index) => (
          <li key={index} className="flex items-center gap-2">
            {index > 0 && <ChevronRight className="h-4 w-4 text-gray-400" />}
            {item.href ? (
              <Link
                to={item.href}
                className="text-gray-500 hover:text-gray-700"
              >
                {index === 0 && <Home className="h-4 w-4" />}
                {index > 0 && item.label}
              </Link>
            ) : (
              <span className="font-medium text-gray-900">{item.label}</span>
            )}
          </li>
        ))}
      </ol>
    </nav>
  )
}

// Uso en AdminUserDetailPage
<Breadcrumbs items={[
  { label: 'Dashboard', href: '/admin' },
  { label: 'Usuarios', href: '/admin/users' },
  { label: user.fullName }
]} />
```

**Complejidad:** Baja-Media | **Impacto:** UX/Navegacion

---

### 12. Verificar Filtrado de Admin en Sidebar
**Archivo:** `acainfo-front/src/shared/components/layout/Sidebar.tsx`

**Problema potencial:** El enlace "Administracion" tiene `roles: ['ADMIN']` pero hay que verificar que funcione correctamente.

**Codigo actual (lineas 188-191):**
```tsx
const filteredNavigation = navigation.filter((item) => {
  if (!item.roles) return true
  return item.roles.some((role) => hasRole(role))
})
```

**Verificacion:** Confirmar que `hasRole('ADMIN')` retorna `false` para usuarios que solo tienen rol STUDENT.

**Complejidad:** Baja | **Impacto:** UX

---

## PRIORIDAD BAJA - Nice-to-have

### 13. Implementar Skeleton Loaders
Reemplazar spinners con skeletons para mejor percepcion de velocidad.

```tsx
// acainfo-front/src/shared/components/ui/Skeleton.tsx
export function Skeleton({ className }: { className?: string }) {
  return (
    <div className={cn('animate-pulse rounded bg-gray-200', className)} />
  )
}

// Ejemplo: TableRowSkeleton
export function TableRowSkeleton({ columns = 5 }: { columns?: number }) {
  return (
    <tr>
      {Array.from({ length: columns }).map((_, i) => (
        <td key={i} className="px-6 py-4">
          <Skeleton className="h-4 w-24" />
        </td>
      ))}
    </tr>
  )
}
```

**Complejidad:** Media | **Impacto:** UX

---

### 14. Mejorar Tablas para Responsive (Cards en Movil)
Convertir tablas a cards en pantallas pequenas para evitar scroll horizontal.

```tsx
// Patron de tabla responsive
<div className="hidden md:block">
  <table>...</table>
</div>
<div className="md:hidden space-y-4">
  {items.map(item => (
    <div key={item.id} className="rounded-lg border bg-white p-4 shadow-sm">
      {/* Contenido en formato card */}
    </div>
  ))}
</div>
```

**Complejidad:** Alta | **Impacto:** UX Movil

---

### 15. Implementar Paginacion en Todas las Tablas
**Paginas sin paginacion:**
- `AdminUsersPage.tsx`
- `AdminEnrollmentsPage.tsx`
- `AdminGroupsPage.tsx`
- `AdminTeachersPage.tsx`
- `AdminSubjectsPage.tsx`
- `AdminSessionsPage.tsx`

**Solucion:** Extraer componente `Pagination.tsx` reutilizable basado en `AdminPaymentsPage`.

**Complejidad:** Media | **Impacto:** Performance

---

### 16. Verificar Contraste de Colores WCAG
**Areas a revisar:**
- Badges amarillos sobre fondo claro
- Texto gris (`text-gray-500`) sobre fondo blanco
- Estados de hover

**Herramienta:** WebAIM Contrast Checker

**Complejidad:** Baja-Media | **Impacto:** Accesibilidad

---

### 17. Migrar Tokens a httpOnly Cookies
**Problema:** Los tokens JWT estan en localStorage, vulnerable a XSS.

**Nota:** Requiere cambios coordinados con backend para implementar cookies httpOnly.

**Complejidad:** Alta | **Impacto:** Seguridad

---

## Archivos Criticos a Modificar

| Archivo | Mejoras |
|---------|---------|
| `src/shared/services/apiClient.ts` | #1 |
| `src/shared/components/layout/Sidebar.tsx` | #4, #8, #12 |
| `src/features/admin/users/components/UserTable.tsx` | #2 |
| `src/features/admin/payments/components/PaymentTable.tsx` | #2 |
| `src/features/enrollments/components/EnrollmentDetailCard.tsx` | #3 |
| `src/features/enrollments/pages/EnrollmentDetailPage.tsx` | #5 |

---

## Nuevos Archivos a Crear

```
acainfo-front/src/shared/
├── components/
│   ├── common/
│   │   ├── ConfirmDialog.tsx      (#5)
│   │   ├── LoadingState.tsx       (#6)
│   │   └── ErrorState.tsx         (#10)
│   └── ui/
│       ├── Spinner.tsx            (#6)
│       ├── Badge.tsx              (#7)
│       ├── Breadcrumbs.tsx        (#11)
│       └── Skeleton.tsx           (#13)
├── hooks/
│   ├── useConfirmDialog.ts        (#5)
│   └── useDebounce.ts             (#9)
└── config/
    └── badgeConfig.ts             (#7)
```

---

## Verificacion del Plan

1. **Build sin errores:** `npm run build` debe completar exitosamente
2. **Tests visuales:** Verificar que los cambios de UI funcionen correctamente
3. **Accesibilidad:** Probar con lector de pantalla los aria-labels
4. **Performance:** Verificar debounce con Network tab del navegador
5. **Roles:** Verificar que usuarios no-admin no vean datos administrativos
6. **Responsive:** Verificar en dispositivos moviles

---

## Estimacion de Esfuerzo

| Prioridad | Items | Esfuerzo Estimado |
|-----------|-------|-------------------|
| **ALTA** | 6 mejoras | 2-3 dias |
| **MEDIA** | 6 mejoras | 4-5 dias |
| **BAJA** | 5 mejoras | 3-4 dias |
| **Total** | **17 mejoras** | **9-12 dias** |

---

## Orden de Implementacion Recomendado

### Fase 1 (Dia 1-2): Seguridad y Quick Wins
1. Eliminar header X-User-Id
2. Ocultar IDs en tablas admin
3. Ocultar codigo de asignatura
4. Agregar aria-labels

### Fase 2 (Dia 3-4): Componentes Base
5. Crear LoadingState/Spinner
6. Crear ConfirmDialog
7. Migrar window.confirm (17 archivos)

### Fase 3 (Dia 5-7): Refactorizacion
8. Crear Badge unificado
9. Extraer SVGs a Lucide
10. Implementar debounce

### Fase 4 (Dia 8-9): UX Improvements
11. ErrorState mejorado
12. Breadcrumbs
13. Verificar filtrado sidebar

### Fase 5 (Dia 10-12): Polish
14. Skeleton loaders
15. Tablas responsive
16. Paginacion global
17. Verificar contraste WCAG

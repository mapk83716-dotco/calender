# Calendario de Hábitos Interactivo

App web para definir hábitos y metas propias, marcar su cumplimiento día a día y ver el progreso en gráficas — sin tocar código.

## Stack

- **React + TypeScript + Vite**
- **Tailwind CSS v4** (tema oscuro)
- **Supabase** (Auth + Postgres con Row Level Security) para guardar datos por usuario
- **Recharts** para las gráficas

## Cómo correrlo

```bash
npm install
npm run dev
```

Abre `http://localhost:5173`. Ya viene configurado un archivo `.env` apuntando a un proyecto de Supabase existente (ver abajo). Si quieres usar tu propio proyecto de Supabase, copia `.env.example` a `.env` y pon tu URL y llave "publishable" (Project Settings → API).

## Base de datos (Supabase)

El proyecto usa 3 tablas, todas con Row Level Security (cada usuario solo ve y edita lo suyo):

- **habits**: hábitos definidos por el usuario (nombre, categoría, color, tipo y valor de meta).
- **habit_logs**: un registro por hábito y día (`completed: true/false`).
- **todos**: agenda de pendientes opcional.

La migración SQL para recrear estas tablas está documentada abajo en "Esquema SQL".

## Estructura del proyecto

```
src/
├── components/     → HabitTable, HabitFormModal, DailyProgressChart, CategoryBreakdown
├── pages/          → Login.tsx, Dashboard.tsx, Stats.tsx, TodoPage.tsx
├── lib/            → supabaseClient.ts, dateUtils.ts
├── hooks/          → useAuth.ts, useHabits.ts, useHabitLogs.ts, useTodos.ts
└── types.ts        → tipos compartidos y constantes (categorías, colores)
```

Cuando pidas un cambio, puedes referirte directamente al archivo (por ejemplo "modifica `HabitTable.tsx`" o "cambia el esquema en `useHabits.ts`").

## Funcionalidades implementadas

- **Login / registro** con Supabase Auth (correo y contraseña).
- **Tabla semanal de hábitos**: filas = hábitos, columnas = días de la semana, checkboxes para marcar cumplimiento. Navegación entre semanas (pasadas y futuras) con botón "Hoy".
- **CRUD de hábitos**: crear, editar y eliminar, con categoría, color y una meta configurable por hábito (% de cumplimiento o número de días, en periodo semanal o mensual).
- **Estadísticas** (por mes, con navegación mes anterior/siguiente):
  - Gráfica de barras del % de cumplimiento diario + línea punteada de tu meta.
  - Desglose de cumplido/fallado por categoría.
  - Lista de los días con menor cumplimiento (<50%).
- **Agenda de pendientes** (opcional): tareas sueltas con fecha, separadas en pendientes/completadas.
- Diseño responsive (se adapta a móvil, tablet y escritorio), tema oscuro.

## Limitaciones actuales

- El registro no verifica correo por defecto (depende de la configuración de tu proyecto de Supabase en Authentication → Settings).
- La "meta diaria promedio" en Estadísticas es un promedio simple de las metas de todos los hábitos activos, normalizado a %; no pondera por importancia.
- No hay edición de tareas de la agenda (solo crear/completar/eliminar).
- El bundle de producción es un solo archivo JS (~245 KB comprimido); para una app más grande convendría dividir en chunks (`import()` dinámico).

## Próximos pasos sugeridos

1. Agregar recordatorios/notificaciones para hábitos no marcados en el día.
2. Permitir reordenar hábitos (drag & drop) y archivarlos en vez de solo eliminarlos.
3. Exportar estadísticas (CSV/PDF) para revisiones mensuales.
4. Vista de calendario mensual tipo grid (como alternativa a la tabla semanal).

## Esquema SQL

```sql
create table public.habits (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references auth.users(id) on delete cascade,
  name text not null,
  category text not null default 'General',
  color text not null default '#8b5cf6',
  goal_type text not null default 'percentage' check (goal_type in ('percentage','count')),
  goal_period text not null default 'week' check (goal_period in ('week','month')),
  goal_value numeric not null default 80,
  archived boolean not null default false,
  sort_order integer not null default 0,
  created_at timestamptz not null default now()
);

create table public.habit_logs (
  id uuid primary key default gen_random_uuid(),
  habit_id uuid not null references public.habits(id) on delete cascade,
  user_id uuid not null references auth.users(id) on delete cascade,
  log_date date not null,
  completed boolean not null default false,
  created_at timestamptz not null default now(),
  unique (habit_id, log_date)
);

create table public.todos (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references auth.users(id) on delete cascade,
  title text not null,
  due_date date,
  completed boolean not null default false,
  created_at timestamptz not null default now()
);

alter table public.habits enable row level security;
alter table public.habit_logs enable row level security;
alter table public.todos enable row level security;

create policy "habits_owner_all" on public.habits
  for all using (auth.uid() = user_id) with check (auth.uid() = user_id);
create policy "habit_logs_owner_all" on public.habit_logs
  for all using (auth.uid() = user_id) with check (auth.uid() = user_id);
create policy "todos_owner_all" on public.todos
  for all using (auth.uid() = user_id) with check (auth.uid() = user_id);
```

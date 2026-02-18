# Design System — Schakel AI

> Visuele standaarden voor alle Schakel-producten. Material Design 3 als basis, aangepast voor data-intensieve B2B applicaties.

## Design Principes

- **Clarity First**: Elk element dient een functioneel doel
- **Data Hierarchy**: Duidelijk visueel onderscheid tussen primaire data en metadata
- **Actionability**: CTAs en interactieve elementen zijn direct herkenbaar
- **Status Transparency**: Pipeline states, errors, en progress zijn instant zichtbaar

## Typografie

**Font Stack**: Google Fonts via CDN
- **Primary**: Inter (headings, UI elementen, data)
- **Monospace**: JetBrains Mono (logs, timestamps, IDs)

**Hiërarchie**:
| Gebruik | Tailwind Classes |
|---------|-----------------|
| Page titles | `text-2xl font-semibold` |
| Section headers | `text-lg font-medium` |
| Body/data | `text-sm font-normal` |
| Labels | `text-xs font-medium uppercase tracking-wide` |
| Monospace data | `text-sm font-mono` |

## Layout

**Spacing**: Tailwind units van 2, 4, 6, 8, 12, 16
- Component padding: `p-4`, `p-6`
- Section spacing: `gap-6`, `gap-8`
- Card spacing: `p-6`
- Table cells: `px-4 py-3`
- Forms: `space-y-4`

**Grid**:
- Container: `max-w-7xl mx-auto px-4`
- Dashboard: Single column, full-width cards
- Detail pages: Two-column (8/4 split)
- Forms: Single column `max-w-2xl`

## Componenten

**Buttons**: `rounded-md`, drie varianten:
- Primary: Filled solid
- Secondary: Outlined
- Tertiary: Ghost/text

**Inputs**: `h-10`, `rounded-md`, clear focus states

**Badges**: `rounded-full px-3 py-1 text-xs` voor status indicators

**Cards**: `shadow-sm border rounded-lg p-6`

**Tables**:
- Striped rows voor scanbaarheid
- Sticky header on scroll
- Sorteerbare kolommen
- Row hover states
- Responsive: stack naar cards op mobile

## Animaties

**Filosofie**: Minimaal en doelgericht.

**Toegestaan**:
- Button hover/active states (subtiele scale/opacity)
- Loading spinners voor async operaties
- Toast slide-in/fade-out
- Smooth collapse/expand

**Verboden**:
- Page transition effects
- Decoratieve animaties
- Scroll-triggered effects
- Complexe micro-interactions

## Accessibility

- Keyboard navigatie: volledig
- ARIA labels: alle icon buttons en interactieve elementen
- Focus states: zichtbare outline
- Contrast: WCAG AA standaard
- Screen reader: correcte heading hierarchy, semantic HTML

## Afbeeldingen

Geen decoratieve beelden. Dit is een utility applicatie. Alleen:
- Logo: SVG, ~32px hoogte
- Icons: Lucide icons (shadcn/ui standaard)
- Empty states: simpel icon + tekst

---

*Laatst bijgewerkt: Februari 2026*

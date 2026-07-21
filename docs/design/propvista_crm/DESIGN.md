---
name: PropVista CRM
colors:
  surface: '#faf8ff'
  surface-dim: '#d9d9e4'
  surface-bright: '#faf8ff'
  surface-container-lowest: '#ffffff'
  surface-container-low: '#f3f3fd'
  surface-container: '#ededf8'
  surface-container-high: '#e7e7f2'
  surface-container-highest: '#e1e2ec'
  on-surface: '#191b23'
  on-surface-variant: '#434654'
  inverse-surface: '#2e3038'
  inverse-on-surface: '#f0f0fb'
  outline: '#737685'
  outline-variant: '#c3c6d6'
  surface-tint: '#0c56d0'
  primary: '#003d9b'
  on-primary: '#ffffff'
  primary-container: '#0052cc'
  on-primary-container: '#c4d2ff'
  inverse-primary: '#b2c5ff'
  secondary: '#873da6'
  on-secondary: '#ffffff'
  secondary-container: '#de8ffd'
  on-secondary-container: '#661a86'
  tertiary: '#7b2600'
  on-tertiary: '#ffffff'
  tertiary-container: '#a33500'
  on-tertiary-container: '#ffc6b2'
  error: '#ba1a1a'
  on-error: '#ffffff'
  error-container: '#ffdad6'
  on-error-container: '#93000a'
  primary-fixed: '#dae2ff'
  primary-fixed-dim: '#b2c5ff'
  on-primary-fixed: '#001848'
  on-primary-fixed-variant: '#0040a2'
  secondary-fixed: '#f8d8ff'
  secondary-fixed-dim: '#ecb2ff'
  on-secondary-fixed: '#320047'
  on-secondary-fixed-variant: '#6c228c'
  tertiary-fixed: '#ffdbcf'
  tertiary-fixed-dim: '#ffb59b'
  on-tertiary-fixed: '#380d00'
  on-tertiary-fixed-variant: '#812800'
  background: '#faf8ff'
  on-background: '#191b23'
  surface-variant: '#e1e2ec'
typography:
  display-lg:
    fontFamily: Inter
    fontSize: 48px
    fontWeight: '700'
    lineHeight: 56px
    letterSpacing: -0.02em
  headline-lg:
    fontFamily: Inter
    fontSize: 32px
    fontWeight: '700'
    lineHeight: 40px
    letterSpacing: -0.01em
  headline-lg-mobile:
    fontFamily: Inter
    fontSize: 24px
    fontWeight: '700'
    lineHeight: 32px
  headline-md:
    fontFamily: Inter
    fontSize: 24px
    fontWeight: '600'
    lineHeight: 32px
  body-lg:
    fontFamily: Inter
    fontSize: 18px
    fontWeight: '400'
    lineHeight: 28px
  body-md:
    fontFamily: Inter
    fontSize: 16px
    fontWeight: '400'
    lineHeight: 24px
  body-sm:
    fontFamily: Inter
    fontSize: 14px
    fontWeight: '400'
    lineHeight: 20px
  label-md:
    fontFamily: Inter
    fontSize: 14px
    fontWeight: '600'
    lineHeight: 16px
    letterSpacing: 0.05em
  label-sm:
    fontFamily: Inter
    fontSize: 12px
    fontWeight: '500'
    lineHeight: 16px
rounded:
  sm: 0.25rem
  DEFAULT: 0.5rem
  md: 0.75rem
  lg: 1rem
  xl: 1.5rem
  full: 9999px
spacing:
  base: 8px
  xs: 4px
  sm: 12px
  md: 16px
  lg: 24px
  xl: 40px
  container-max: 1440px
  gutter: 24px
  margin-mobile: 16px
---

## Brand & Style

The design system is engineered for a high-performance PropTech environment, prioritizing clarity, trust, and intelligence. The brand personality is professional yet innovative, balancing the traditional stability of real estate with the cutting-edge efficiency of artificial intelligence.

The visual style follows a **Corporate / Modern** aesthetic with a strong emphasis on **Minimalism**. It utilizes a "Utility-First" approach: standard CRM operations are quiet and functional, while AI-enhanced insights are highlighted with a distinct secondary accent. The interface avoids unnecessary decoration, relying instead on precise alignment, generous whitespace, and a high-contrast typographic hierarchy to guide the user through complex property data and client relationships.

## Colors

The palette is strictly partitioned to manage user cognitive load. 

- **Primary (#0052CC):** Used for core system actions, primary buttons, and navigational links. It represents the "human" agency within the CRM.
- **Secondary / AI (#8E44AD):** This color is reserved exclusively for AI-driven features. This includes match-score badges, automated data parsing indicators, and the virtual assistant interface. It must never be used for standard system actions.
- **Neutrals:** The foundation is built on #F5F5F7 for page backgrounds to reduce eye strain, while #FFFFFF is used for "Surface" elements like cards. #1A1A1A provides high-legibility for text and primary icons.

## Typography

This design system utilizes **Inter** for all roles to maintain a systematic, utilitarian feel. The hierarchy is driven by weight and scale. 

- **Headlines:** Use Bold (700) or SemiBold (600) weights with slight negative letter-spacing for a compact, authoritative look.
- **Body Text:** Standardized at 16px for optimal readability. Use 14px for secondary metadata or dense data tables.
- **AI Labels:** When typography is used within AI contexts (like a "Match Score"), it should often be paired with the Secondary color and Medium/SemiBold weights to ensure it is distinguished from static data.

## Layout & Spacing

The layout follows a **Fluid Grid** model based on an 8px base unit. 

- **Desktop:** 12-column grid with 24px gutters and 40px side margins. 
- **Tablet:** 8-column grid with 24px gutters and 24px side margins.
- **Mobile:** 4-column grid with 16px gutters and 16px side margins.

Content should be grouped into logical "sections" using 40px (xl) vertical spacing. Inside cards and components, 16px (md) and 24px (lg) are the default padding values to ensure the "generous whitespace" required for a premium aesthetic.

## Elevation & Depth

Hierarchy is established through **Tonal Layers** and **Ambient Shadows**. 

1. **Level 0 (Background):** #F5F5F7 — The lowest layer.
2. **Level 1 (Cards/Surfaces):** #FFFFFF — Elevated via a very subtle, diffused shadow: `0px 2px 4px rgba(0, 0, 0, 0.05)`.
3. **Level 2 (Dropdowns/Modals):** Elevated further with a more pronounced shadow: `0px 12px 24px rgba(0, 0, 0, 0.1)`.

Borders are used sparingly to define structure without adding visual noise. A 1px solid border in #E5E5EA is the standard for non-elevated containers like input fields or table rows.

## Shapes

The design system uses a "Rounded" geometry (Level 2). 

- **Standard Components:** Buttons, Input fields, and small UI elements use a 0.5rem (8px) radius.
- **Cards:** To meet the specific brand requirement, cards and larger containers use a 10px (0.625rem) radius.
- **Interactive Elements:** Active states or focus rings should follow the underlying component's radius exactly.

## Components

- **Buttons:** 
  - *Primary:* Solid #0052CC with white text. 
  - *Secondary:* Outline #E5E5EA with #1A1A1A text, or Ghost (no border/background).
  - *AI Action:* Solid #8E44AD (use only for AI-specific triggers).
- **Cards:** White background, 10px corner radius, 24px internal padding, and the "Level 1" shadow.
- **Input Fields:** 8px radius, #E5E5EA border. On focus, use a 2px #0052CC border.
- **AI Match Chips:** Small badges with a light #8E44AD tint background (10% opacity) and solid #8E44AD text to highlight match percentages.
- **Lists:** Clean rows with 1px #E5E5EA bottom borders. 16px vertical padding per row for high density with clarity.
- **Checkboxes/Radios:** Use the Primary Blue for selected states. For AI-suggested selections, use the Secondary Purple.
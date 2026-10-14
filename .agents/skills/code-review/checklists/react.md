# Code Review Checklist – React

- [ ] Components are pure and functionally written where possible (no unnecessary class components).
- [ ] Props and state are typed and validated (using PropTypes or TypeScript).
- [ ] No direct DOM manipulation unless necessary (prefer refs & state).
- [ ] Proper use of useEffect/useMemo/useCallback for side effects and performance optimizations.
- [ ] No missing dependencies in useEffect/useMemo/useCallback dependency arrays.
- [ ] Keys are set for list-rendered components.
- [ ] No unnecessary re-renders (check memoization and dependencies).
- [ ] Handlers and event listeners are not recreated on each render.
- [ ] Prop drilling is minimized (prefer context or state managers if deeply nested).
- [ ] Avoid inline functions/objects in render where possible.
- [ ] Components are structured in small, focused units – SRP followed.
- [ ] UI logic is separated from data-fetching/business logic.
- [ ] All user input is sanitized and validated.
- [ ] Strict mode warnings are addressed.
- [ ] Test coverage for interactive and critical components.
- [ ] Accessibility checks (labels, keyboard navigation, aria-* attributes).
- [ ] Proper error handling for async operations.
- [ ] No hardcoded styles—either use CSS modules, styled-components, etc.

---

For advanced projects:
- [ ] Lazy/Suspense used for code splitting where appropriate.
- [ ] Global state usage (Redux, Zustand, etc.) is justified—no unnecessary global state.

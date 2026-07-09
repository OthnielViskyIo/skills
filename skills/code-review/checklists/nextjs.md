# Code Review Checklist – Next.js

- [ ] Determine whether the project uses Pages Router (`/pages`) or App Router (`/app`) before applying routing-specific checks.
- [ ] Pages and API routes are correctly placed under `/pages` or `/app` (for App Router).
- [ ] Server/client components are properly distinguished (if using App Router).
- [ ] Data fetching uses the right method (getServerSideProps, getStaticProps, getInitialProps, or server actions).
- [ ] API routes validate incoming data and handle errors securely.
- [ ] No sensitive data is leaked to the client.
- [ ] Dynamic imports and code splitting are leveraged for large/lazy loaded components.
- [ ] SEO best practices followed: <Head> tags set, open graph, meta tags configured.
- [ ] Routing is idiomatic; use Link, not a-tags for navigation.
- [ ] Proper error pages (404, 500) are implemented.
- [ ] No deprecated Next.js APIs.
- [ ] Images use the <Image> component and are optimized.
- [ ] Environment variables access is safe—never expose server secrets to the browser.

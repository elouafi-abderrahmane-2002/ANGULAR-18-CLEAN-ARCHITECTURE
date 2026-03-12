# 🏗️ Angular 18 — Clean Architecture & JWT

Un projet Angular qui "marche" est facile à faire. Un projet Angular qu'un
autre développeur peut reprendre, modifier, étendre six mois plus tard sans
tout casser — c'est une autre histoire. Ce projet applique une architecture
modulaire stricte, avec une séparation claire des responsabilités, un système
JWT complet et une base conçue pour durer.

---

## Architecture modulaire — structure des dossiers

```
  src/app/
  │
  ├── core/                     ← services singleton, guards, interceptors
  │   ├── guards/
  │   │   ├── auth.guard.ts         ← redirige si non authentifié
  │   │   └── role.guard.ts         ← vérifie le rôle (ADMIN, USER, etc.)
  │   ├── interceptors/
  │   │   ├── jwt.interceptor.ts    ← injecte le token dans chaque requête
  │   │   └── error.interceptor.ts  ← gestion centralisée des erreurs HTTP
  │   ├── services/
  │   │   └── auth.service.ts       ← login, logout, refresh token
  │   └── core.module.ts            ← forRoot guard : importé une seule fois
  │
  ├── features/                 ← modules fonctionnels, lazy-loadés
  │   ├── dashboard/
  │   │   ├── components/
  │   │   ├── services/
  │   │   └── dashboard.module.ts
  │   └── settings/
  │       ├── components/
  │       └── settings.module.ts
  │
  ├── shared/                   ← composants/pipes/directives réutilisables
  │   ├── components/
  │   │   ├── navbar/
  │   │   └── spinner/
  │   ├── directives/
  │   │   └── has-role.directive.ts  ← *hasRole="'ADMIN'"
  │   ├── pipes/
  │   └── shared.module.ts
  │
  └── app-routing.module.ts     ← routing principal avec lazy loading
```

---

## Lazy loading — routing configuré

```typescript
// app-routing.module.ts
const routes: Routes = [
  {
    path: 'dashboard',
    canActivate: [AuthGuard],
    loadChildren: () =>
      import('./features/dashboard/dashboard.module')
        .then(m => m.DashboardModule)
  },
  {
    path: 'admin',
    canActivate: [AuthGuard, RoleGuard],
    data: { roles: ['ADMIN'] },
    loadChildren: () =>
      import('./features/admin/admin.module')
        .then(m => m.AdminModule)
  },
  {
    path: 'auth',
    loadChildren: () =>
      import('./features/auth/auth.module')
        .then(m => m.AuthModule)
  },
  { path: '', redirectTo: '/dashboard', pathMatch: 'full' },
  { path: '**', redirectTo: '/auth/login' }
];

// Résultat : le bundle initial ne charge que ce qui est nécessaire.
// Les modules Feature sont chargés à la demande — première navigation
// vers /dashboard = chargement de DashboardModule uniquement.
```

---

## JWT Interceptor — injection automatique du token

```typescript
// core/interceptors/jwt.interceptor.ts
import { Injectable } from '@angular/core';
import {
  HttpRequest, HttpHandler, HttpEvent,
  HttpInterceptor, HttpErrorResponse
} from '@angular/common/http';
import { Observable, throwError, BehaviorSubject } from 'rxjs';
import { catchError, filter, take, switchMap } from 'rxjs/operators';
import { AuthService } from '../services/auth.service';

@Injectable()
export class JwtInterceptor implements HttpInterceptor {

  private isRefreshing = false;
  private refreshTokenSubject = new BehaviorSubject<string | null>(null);

  constructor(private authService: AuthService) {}

  intercept(req: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
    const token = this.authService.getAccessToken();

    // Injecter le token si présent
    const authReq = token ? this.addToken(req, token) : req;

    return next.handle(authReq).pipe(
      catchError(error => {
        if (error instanceof HttpErrorResponse && error.status === 401) {
          return this.handle401(req, next);
        }
        return throwError(() => error);
      })
    );
  }

  private addToken(req: HttpRequest<any>, token: string): HttpRequest<any> {
    return req.clone({
      setHeaders: { Authorization: `Bearer ${token}` }
    });
  }

  private handle401(req: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
    if (!this.isRefreshing) {
      this.isRefreshing = true;
      this.refreshTokenSubject.next(null);

      return this.authService.refreshToken().pipe(
        switchMap(tokens => {
          this.isRefreshing = false;
          this.refreshTokenSubject.next(tokens.accessToken);
          return next.handle(this.addToken(req, tokens.accessToken));
        }),
        catchError(err => {
          this.isRefreshing = false;
          this.authService.logout();
          return throwError(() => err);
        })
      );
    }

    // Mettre les requêtes en attente pendant le refresh
    return this.refreshTokenSubject.pipe(
      filter(token => token !== null),
      take(1),
      switchMap(token => next.handle(this.addToken(req, token!)))
    );
  }
}
```

---

## Route Guard par rôle

```typescript
// core/guards/role.guard.ts
@Injectable({ providedIn: 'root' })
export class RoleGuard implements CanActivate {

  constructor(private authService: AuthService, private router: Router) {}

  canActivate(route: ActivatedRouteSnapshot): boolean {
    const requiredRoles: string[] = route.data['roles'] ?? [];
    const userRoles = this.authService.getUserRoles();

    const hasAccess = requiredRoles.some(role => userRoles.includes(role));

    if (!hasAccess) {
      this.router.navigate(['/unauthorized']);
      return false;
    }
    return true;
  }
}
```

---

## Ce que j'ai appris

Le `BehaviorSubject` dans l'interceptor de refresh token est essentiel.
Sans lui, si 3 requêtes simultanées reçoivent un 401, on déclenche 3 refresh
simultanés — ce qui invalide les tokens et déconnecte l'utilisateur.

Avec le pattern ci-dessus, seul le premier 401 déclenche le refresh.
Les suivants attendent que le `refreshTokenSubject` émette le nouveau token,
puis repartent avec. C'est un problème de concurrence qui ne se voit
qu'en conditions réelles — pas en développement local avec une seule requête.

---

*Projet réalisé dans le cadre de ma formation ingénieur — ENSET Mohammedia*
*Par **Abderrahmane Elouafi** · [LinkedIn](https://www.linkedin.com/in/abderrahmane-elouafi-43226736b/) · [Portfolio](https://my-first-porfolio-six.vercel.app/)*

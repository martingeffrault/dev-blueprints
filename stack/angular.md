# Angular (2025)

> **Last updated**: January 2026
> **Versions covered**: Angular 18+, 19, 20, 21
> **Purpose**: Enterprise-grade frontend framework with TypeScript

---

## Philosophy (2025-2026)

Angular in 2025 is **signals-first and standalone-first**. The framework has embraced fine-grained reactivity, leaving Zone.js behind for most use cases. Standalone components are now the default.

**Key shifts:**
- **Signals everywhere** — Stable in Angular 20+, replace most RxJS
- **Standalone components** — No NgModules required
- **Zoneless Angular** — Better performance, simpler mental model
- **linkedSignal** — Derived state with user overrides
- **Angular Aria** — Accessibility-first UI primitives
- **Signal Forms** — Next-gen reactive forms (preview)

---

## TL;DR

- Use standalone components exclusively
- Use Signals for local state and derivations
- Keep RxJS for async streams and complex pipelines
- Use `ChangeDetectionStrategy.OnPush` everywhere
- Use `inject()` function over constructor injection
- Lazy load routes and feature components
- Use typed forms and strict template checking
- Write explicit types, even when inferable

---

## Best Practices

### Project Structure (Feature-Based)

```
src/
├── app/
│   ├── app.component.ts
│   ├── app.config.ts
│   ├── app.routes.ts
│   ├── core/                    # Singleton services
│   │   ├── services/
│   │   │   ├── auth.service.ts
│   │   │   └── api.service.ts
│   │   ├── interceptors/
│   │   │   └── auth.interceptor.ts
│   │   ├── guards/
│   │   │   └── auth.guard.ts
│   │   └── index.ts
│   ├── shared/                  # Reusable components
│   │   ├── components/
│   │   │   ├── button/
│   │   │   └── modal/
│   │   ├── directives/
│   │   ├── pipes/
│   │   └── index.ts
│   ├── features/                # Feature modules
│   │   ├── auth/
│   │   │   ├── login/
│   │   │   │   └── login.component.ts
│   │   │   ├── register/
│   │   │   │   └── register.component.ts
│   │   │   ├── services/
│   │   │   │   └── auth-api.service.ts
│   │   │   ├── store/
│   │   │   │   └── auth.store.ts
│   │   │   └── auth.routes.ts
│   │   ├── dashboard/
│   │   │   ├── dashboard.component.ts
│   │   │   └── dashboard.routes.ts
│   │   └── users/
│   │       ├── user-list/
│   │       ├── user-detail/
│   │       ├── services/
│   │       ├── store/
│   │       └── users.routes.ts
│   └── layouts/
│       ├── main-layout/
│       └── auth-layout/
├── environments/
│   ├── environment.ts
│   └── environment.prod.ts
└── styles/
```

### App Configuration

```typescript
// src/app/app.config.ts
import { ApplicationConfig, provideZoneChangeDetection } from '@angular/core';
import { provideRouter, withComponentInputBinding } from '@angular/router';
import { provideHttpClient, withInterceptors } from '@angular/common/http';
import { provideAnimationsAsync } from '@angular/platform-browser/animations/async';

import { routes } from './app.routes';
import { authInterceptor } from './core/interceptors/auth.interceptor';
import { errorInterceptor } from './core/interceptors/error.interceptor';

export const appConfig: ApplicationConfig = {
  providers: [
    provideZoneChangeDetection({ eventCoalescing: true }),
    provideRouter(routes, withComponentInputBinding()),
    provideHttpClient(
      withInterceptors([authInterceptor, errorInterceptor])
    ),
    provideAnimationsAsync(),
  ],
};
```

### Routes with Lazy Loading

```typescript
// src/app/app.routes.ts
import { Routes } from '@angular/router';
import { authGuard } from './core/guards/auth.guard';

export const routes: Routes = [
  {
    path: '',
    loadComponent: () =>
      import('./layouts/main-layout/main-layout.component').then(
        (m) => m.MainLayoutComponent
      ),
    canActivate: [authGuard],
    children: [
      {
        path: '',
        loadComponent: () =>
          import('./features/dashboard/dashboard.component').then(
            (m) => m.DashboardComponent
          ),
      },
      {
        path: 'users',
        loadChildren: () =>
          import('./features/users/users.routes').then((m) => m.USERS_ROUTES),
      },
    ],
  },
  {
    path: 'auth',
    loadChildren: () =>
      import('./features/auth/auth.routes').then((m) => m.AUTH_ROUTES),
  },
  {
    path: '**',
    loadComponent: () =>
      import('./shared/components/not-found/not-found.component').then(
        (m) => m.NotFoundComponent
      ),
  },
];
```

### Standalone Component with Signals

```typescript
// src/app/features/users/user-list/user-list.component.ts
import {
  Component,
  computed,
  inject,
  signal,
  ChangeDetectionStrategy,
} from '@angular/core';
import { CommonModule } from '@angular/common';
import { FormsModule } from '@angular/forms';
import { RouterLink } from '@angular/router';
import { toSignal } from '@angular/core/rxjs-interop';

import { UsersService } from '../services/users.service';
import { UserCardComponent } from '../user-card/user-card.component';
import { SearchInputComponent } from '@shared/components/search-input/search-input.component';
import { LoadingSpinnerComponent } from '@shared/components/loading-spinner/loading-spinner.component';
import type { User } from '../models/user.model';

@Component({
  selector: 'app-user-list',
  standalone: true,
  imports: [
    CommonModule,
    FormsModule,
    RouterLink,
    UserCardComponent,
    SearchInputComponent,
    LoadingSpinnerComponent,
  ],
  template: `
    <div class="user-list">
      <header class="user-list__header">
        <h1>Users</h1>
        <app-search-input
          [value]="searchTerm()"
          (valueChange)="searchTerm.set($event)"
          placeholder="Search users..."
        />
      </header>

      @if (isLoading()) {
        <app-loading-spinner />
      } @else if (error()) {
        <div class="error-message">
          {{ error() }}
          <button (click)="retry()">Retry</button>
        </div>
      } @else {
        <div class="user-list__grid">
          @for (user of filteredUsers(); track user.id) {
            <app-user-card
              [user]="user"
              (select)="onUserSelect($event)"
            />
          } @empty {
            <p class="no-results">No users found</p>
          }
        </div>
      }
    </div>
  `,
  styleUrl: './user-list.component.scss',
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class UserListComponent {
  private readonly usersService = inject(UsersService);

  // Local state
  searchTerm = signal<string>('');

  // Server state as signal
  private readonly usersResource = this.usersService.getUsers();
  users = this.usersResource.value;
  isLoading = this.usersResource.isLoading;
  error = this.usersResource.error;

  // Computed (derived) state
  filteredUsers = computed(() => {
    const term = this.searchTerm().toLowerCase();
    const users = this.users() ?? [];

    if (!term) return users;

    return users.filter(
      (user) =>
        user.name.toLowerCase().includes(term) ||
        user.email.toLowerCase().includes(term)
    );
  });

  onUserSelect(user: User): void {
    console.log('Selected user:', user);
  }

  retry(): void {
    this.usersResource.reload();
  }
}
```

### Signal-Based Service

```typescript
// src/app/features/users/services/users.service.ts
import { Injectable, inject, signal, computed } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { rxResource } from '@angular/core/rxjs-interop';
import { environment } from '@env/environment';
import type { User, CreateUserDto, UpdateUserDto } from '../models/user.model';

@Injectable({ providedIn: 'root' })
export class UsersService {
  private readonly http = inject(HttpClient);
  private readonly apiUrl = `${environment.apiUrl}/users`;

  // For components that need resource-based loading
  getUsers() {
    return rxResource({
      loader: () => this.http.get<User[]>(this.apiUrl),
    });
  }

  getUserById(id: string) {
    return rxResource({
      request: () => id,
      loader: ({ request: userId }) =>
        this.http.get<User>(`${this.apiUrl}/${userId}`),
    });
  }

  // For imperative operations
  createUser(dto: CreateUserDto) {
    return this.http.post<User>(this.apiUrl, dto);
  }

  updateUser(id: string, dto: UpdateUserDto) {
    return this.http.patch<User>(`${this.apiUrl}/${id}`, dto);
  }

  deleteUser(id: string) {
    return this.http.delete<void>(`${this.apiUrl}/${id}`);
  }
}
```

### Signal Store (State Management)

```typescript
// src/app/features/users/store/users.store.ts
import { computed, inject } from '@angular/core';
import {
  signalStore,
  withState,
  withComputed,
  withMethods,
  patchState,
} from '@ngrx/signals';
import { rxMethod } from '@ngrx/signals/rxjs-interop';
import { pipe, switchMap, tap } from 'rxjs';
import { tapResponse } from '@ngrx/operators';

import { UsersService } from '../services/users.service';
import type { User } from '../models/user.model';

interface UsersState {
  users: User[];
  selectedUserId: string | null;
  isLoading: boolean;
  error: string | null;
}

const initialState: UsersState = {
  users: [],
  selectedUserId: null,
  isLoading: false,
  error: null,
};

export const UsersStore = signalStore(
  { providedIn: 'root' },
  withState(initialState),
  withComputed((store) => ({
    selectedUser: computed(() =>
      store.users().find((u) => u.id === store.selectedUserId())
    ),
    userCount: computed(() => store.users().length),
  })),
  withMethods((store, usersService = inject(UsersService)) => ({
    selectUser(id: string): void {
      patchState(store, { selectedUserId: id });
    },

    loadUsers: rxMethod<void>(
      pipe(
        tap(() => patchState(store, { isLoading: true, error: null })),
        switchMap(() =>
          usersService.getUsers().value$.pipe(
            tapResponse({
              next: (users) => patchState(store, { users, isLoading: false }),
              error: (error: Error) =>
                patchState(store, {
                  error: error.message,
                  isLoading: false,
                }),
            })
          )
        )
      )
    ),
  }))
);
```

### linkedSignal Example

```typescript
// src/app/features/settings/settings.component.ts
import { Component, linkedSignal, signal } from '@angular/core';

interface Option {
  id: string;
  label: string;
}

@Component({
  selector: 'app-settings',
  standalone: true,
  template: `
    <div class="settings">
      <label>
        Category:
        <select [value]="selectedCategory()" (change)="onCategoryChange($event)">
          @for (cat of categories(); track cat.id) {
            <option [value]="cat.id">{{ cat.label }}</option>
          }
        </select>
      </label>

      <label>
        Option:
        <select [value]="selectedOption()" (change)="selectedOption.set($any($event.target).value)">
          @for (opt of options(); track opt.id) {
            <option [value]="opt.id">{{ opt.label }}</option>
          }
        </select>
      </label>
    </div>
  `,
})
export class SettingsComponent {
  // Source signals
  categories = signal<Option[]>([
    { id: 'general', label: 'General' },
    { id: 'privacy', label: 'Privacy' },
    { id: 'notifications', label: 'Notifications' },
  ]);

  selectedCategory = signal<string>('general');

  // Options that depend on category
  options = computed(() => {
    const category = this.selectedCategory();
    return this.getOptionsForCategory(category);
  });

  // linkedSignal: auto-resets when options change, but user can override
  selectedOption = linkedSignal(() => this.options()[0]?.id ?? '');

  onCategoryChange(event: Event): void {
    const value = (event.target as HTMLSelectElement).value;
    this.selectedCategory.set(value);
    // selectedOption automatically resets to first option!
  }

  private getOptionsForCategory(category: string): Option[] {
    const optionsMap: Record<string, Option[]> = {
      general: [
        { id: 'theme', label: 'Theme' },
        { id: 'language', label: 'Language' },
      ],
      privacy: [
        { id: 'cookies', label: 'Cookies' },
        { id: 'tracking', label: 'Tracking' },
      ],
      notifications: [
        { id: 'email', label: 'Email' },
        { id: 'push', label: 'Push' },
      ],
    };
    return optionsMap[category] ?? [];
  }
}
```

### Functional HTTP Interceptor

```typescript
// src/app/core/interceptors/auth.interceptor.ts
import { HttpInterceptorFn } from '@angular/common/http';
import { inject } from '@angular/core';
import { AuthService } from '../services/auth.service';

export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const authService = inject(AuthService);
  const token = authService.token();

  if (token) {
    const clonedReq = req.clone({
      setHeaders: {
        Authorization: `Bearer ${token}`,
      },
    });
    return next(clonedReq);
  }

  return next(req);
};

// src/app/core/interceptors/error.interceptor.ts
import { HttpInterceptorFn, HttpErrorResponse } from '@angular/common/http';
import { inject } from '@angular/core';
import { Router } from '@angular/router';
import { catchError, throwError } from 'rxjs';
import { AuthService } from '../services/auth.service';

export const errorInterceptor: HttpInterceptorFn = (req, next) => {
  const router = inject(Router);
  const authService = inject(AuthService);

  return next(req).pipe(
    catchError((error: HttpErrorResponse) => {
      if (error.status === 401) {
        authService.logout();
        router.navigate(['/auth/login']);
      }
      return throwError(() => error);
    })
  );
};
```

### Functional Route Guard

```typescript
// src/app/core/guards/auth.guard.ts
import { inject } from '@angular/core';
import { CanActivateFn, Router } from '@angular/router';
import { AuthService } from '../services/auth.service';

export const authGuard: CanActivateFn = () => {
  const authService = inject(AuthService);
  const router = inject(Router);

  if (authService.isAuthenticated()) {
    return true;
  }

  return router.createUrlTree(['/auth/login']);
};
```

### Typed Reactive Forms

```typescript
// src/app/features/auth/login/login.component.ts
import { Component, inject } from '@angular/core';
import { CommonModule } from '@angular/common';
import {
  FormBuilder,
  ReactiveFormsModule,
  Validators,
} from '@angular/forms';
import { Router } from '@angular/router';

import { AuthService } from '@core/services/auth.service';

@Component({
  selector: 'app-login',
  standalone: true,
  imports: [CommonModule, ReactiveFormsModule],
  template: `
    <form [formGroup]="form" (ngSubmit)="onSubmit()">
      <div class="form-group">
        <label for="email">Email</label>
        <input
          id="email"
          type="email"
          formControlName="email"
          [class.invalid]="form.controls.email.invalid && form.controls.email.touched"
        />
        @if (form.controls.email.errors?.['required'] && form.controls.email.touched) {
          <span class="error">Email is required</span>
        }
        @if (form.controls.email.errors?.['email'] && form.controls.email.touched) {
          <span class="error">Invalid email format</span>
        }
      </div>

      <div class="form-group">
        <label for="password">Password</label>
        <input
          id="password"
          type="password"
          formControlName="password"
        />
        @if (form.controls.password.errors?.['required'] && form.controls.password.touched) {
          <span class="error">Password is required</span>
        }
        @if (form.controls.password.errors?.['minlength'] && form.controls.password.touched) {
          <span class="error">Password must be at least 8 characters</span>
        }
      </div>

      <button type="submit" [disabled]="form.invalid || isLoading()">
        {{ isLoading() ? 'Logging in...' : 'Login' }}
      </button>
    </form>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class LoginComponent {
  private readonly fb = inject(FormBuilder);
  private readonly authService = inject(AuthService);
  private readonly router = inject(Router);

  isLoading = signal(false);

  // Typed form - TypeScript knows the shape!
  form = this.fb.nonNullable.group({
    email: ['', [Validators.required, Validators.email]],
    password: ['', [Validators.required, Validators.minLength(8)]],
  });

  onSubmit(): void {
    if (this.form.invalid) return;

    this.isLoading.set(true);

    // form.getRawValue() returns typed object
    const { email, password } = this.form.getRawValue();

    this.authService.login(email, password).subscribe({
      next: () => {
        this.router.navigate(['/']);
      },
      error: (error) => {
        console.error('Login failed:', error);
        this.isLoading.set(false);
      },
    });
  }
}
```

### Component with Input/Output

```typescript
// src/app/shared/components/user-card/user-card.component.ts
import {
  Component,
  input,
  output,
  ChangeDetectionStrategy,
} from '@angular/core';
import { CommonModule } from '@angular/common';
import type { User } from '@features/users/models/user.model';

@Component({
  selector: 'app-user-card',
  standalone: true,
  imports: [CommonModule],
  template: `
    <div class="user-card" (click)="select.emit(user())">
      <img [src]="user().avatar" [alt]="user().name" class="user-card__avatar" />
      <div class="user-card__info">
        <h3>{{ user().name }}</h3>
        <p>{{ user().email }}</p>
        @if (user().role === 'admin') {
          <span class="badge badge--admin">Admin</span>
        }
      </div>
    </div>
  `,
  styleUrl: './user-card.component.scss',
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class UserCardComponent {
  // Signal-based inputs (Angular 17+)
  user = input.required<User>();

  // Output using output() function
  select = output<User>();
}
```

### Testing

```typescript
// src/app/features/users/user-list/user-list.component.spec.ts
import { ComponentFixture, TestBed } from '@angular/core/testing';
import { provideHttpClient } from '@angular/common/http';
import { provideHttpClientTesting } from '@angular/common/http/testing';
import { signal } from '@angular/core';

import { UserListComponent } from './user-list.component';
import { UsersService } from '../services/users.service';

describe('UserListComponent', () => {
  let component: UserListComponent;
  let fixture: ComponentFixture<UserListComponent>;

  const mockUsers = [
    { id: '1', name: 'John Doe', email: 'john@example.com' },
    { id: '2', name: 'Jane Smith', email: 'jane@example.com' },
  ];

  const mockUsersService = {
    getUsers: () => ({
      value: signal(mockUsers),
      isLoading: signal(false),
      error: signal(null),
      reload: jasmine.createSpy('reload'),
    }),
  };

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [UserListComponent],
      providers: [
        provideHttpClient(),
        provideHttpClientTesting(),
        { provide: UsersService, useValue: mockUsersService },
      ],
    }).compileComponents();

    fixture = TestBed.createComponent(UserListComponent);
    component = fixture.componentInstance;
    fixture.detectChanges();
  });

  it('should create', () => {
    expect(component).toBeTruthy();
  });

  it('should filter users by search term', () => {
    component.searchTerm.set('john');
    expect(component.filteredUsers().length).toBe(1);
    expect(component.filteredUsers()[0].name).toBe('John Doe');
  });

  it('should return all users when search term is empty', () => {
    component.searchTerm.set('');
    expect(component.filteredUsers().length).toBe(2);
  });
});
```

---

## Anti-Patterns

### ❌ Using NgModules for New Code

```typescript
// ❌ DON'T — NgModules are legacy
@NgModule({
  declarations: [MyComponent],
  imports: [CommonModule],
})
export class MyModule {}

// ✅ DO — Standalone components
@Component({
  standalone: true,
  imports: [CommonModule],
})
export class MyComponent {}
```

### ❌ Constructor Injection (New Code)

```typescript
// ❌ DON'T — Verbose constructor
constructor(
  private readonly userService: UserService,
  private readonly router: Router,
) {}

// ✅ DO — inject() function
private readonly userService = inject(UserService);
private readonly router = inject(Router);
```

### ❌ Default Change Detection

```typescript
// ❌ DON'T — Default triggers on every event
@Component({
  selector: 'app-list',
})
export class ListComponent {}

// ✅ DO — OnPush with signals
@Component({
  selector: 'app-list',
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class ListComponent {}
```

### ❌ Manual Subscriptions Without Cleanup

```typescript
// ❌ DON'T — Memory leak
ngOnInit() {
  this.http.get('/api/data').subscribe(data => {
    this.data = data;
  });
}

// ✅ DO — Use signals/toSignal or takeUntilDestroyed
private readonly destroyRef = inject(DestroyRef);

ngOnInit() {
  this.http.get('/api/data')
    .pipe(takeUntilDestroyed(this.destroyRef))
    .subscribe(data => {
      this.data.set(data);
    });
}

// Or better — use rxResource
data = rxResource({
  loader: () => this.http.get('/api/data'),
});
```

---

## 2025-2026 Changelog

| Version | Date | Key Changes |
|---------|------|-------------|
| v18 | May 2024 | Zoneless preview, Material 3 |
| v19 | Nov 2024 | linkedSignal, resource API, zoneless SSR |
| **v20** | **May 2025** | **Signals STABLE**, Signal Forms preview |
| **v20.2** | **Aug 2025** | **Zoneless STABLE** |
| **v21** | **Nov 2025** | **Zoneless DEFAULT** for new apps, Angular Aria |

### Angular 21: Zoneless is Now Default

Angular 21 introduces **zoneless as the default** for new applications:

```typescript
// Angular 21+ — New projects are zoneless by default
// No Zone.js in bundle, no zone.js polyfill needed

// app.config.ts (generated by CLI)
export const appConfig: ApplicationConfig = {
  providers: [
    provideExperimentalZonelessChangeDetection(), // Now stable!
    // No provideZoneChangeDetection() needed
  ],
};
```

**Migration schematics available:**
```bash
# Migrate existing project to zoneless
ng generate @angular/core:zoneless
```

### Zoneless Performance Benefits

- **60% faster startup time** with zoneless change detection
- **Smoother applications** when combined with signals
- **80% smaller bundles** possible with lazy loading + zoneless

### Signals Now Stable (v20+)

All fundamental reactivity primitives graduated to stable:
- `signal()` — Reactive state
- `computed()` — Derived state
- `effect()` — Side effects
- `linkedSignal()` — Derived with override capability
- `input()` / `output()` — Signal-based component API
- Signal queries (`viewChild`, `contentChildren`, etc.)

### Testing in Zoneless Mode

```typescript
// Zoneless testing best practice
describe('UserComponent (zoneless)', () => {
  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [UserComponent],
      providers: [
        provideExperimentalZonelessChangeDetection(),
      ],
    }).compileComponents();
  });

  it('should update on signal change', async () => {
    const fixture = TestBed.createComponent(UserComponent);
    const component = fixture.componentInstance;

    // ❌ DON'T — Avoid manual detectChanges in zoneless
    // fixture.detectChanges();

    // ✅ DO — Wait for Angular's automatic change detection
    component.name.set('John');
    await fixture.whenStable();

    expect(fixture.nativeElement.textContent).toContain('John');
  });
});
```

### Signal Best Practices (Performance)

```typescript
// ❌ DON'T — Reference changes trigger unnecessary updates
const user = signal({ name: 'John', age: 30 });
user.update(u => ({ ...u, age: 31 })); // New object every time

// ✅ DO — Break into smaller signals
const userName = signal('John');
const userAge = signal(30);
userAge.set(31); // Only age subscribers update

// ✅ DO — Use computed for derived values
const isAdult = computed(() => userAge() >= 18);
```

### Key Updates

- **Signals stable** — signal, effect, linkedSignal graduated
- **rxResource** — Reactive data loading primitive
- **Signal inputs/outputs** — input(), output(), model()
- **Control flow** — @if, @for, @switch replace *ngIf, *ngFor
- **Zoneless stable** — Production-ready in v20.2+
- **Angular Aria** — Accessibility-first component library

---

## Quick Reference

| Import | Purpose |
|--------|---------|
| `signal()` | Create writable signal |
| `computed()` | Derived signal |
| `effect()` | Side effects on signal changes |
| `linkedSignal()` | Derived with override |
| `input()` | Signal-based input |
| `output()` | Type-safe output |
| `toSignal()` | Observable → Signal |
| `toObservable()` | Signal → Observable |

| Command | Purpose |
|---------|---------|
| `ng new my-app` | Create project |
| `ng g c features/users/user-list` | Generate component |
| `ng g s core/services/auth` | Generate service |
| `ng serve` | Dev server |
| `ng build` | Production build |
| `ng test` | Run tests |

---

## Resources

- [Angular Documentation](https://angular.dev/)
- [Angular Signals Guide](https://angular.dev/guide/signals)
- [Angular Roadmap](https://angular.dev/roadmap)
- [NgRx Signal Store](https://ngrx.io/guide/signals)
- [Angular 2025 Coding Standards](https://dev.to/codewithrajat/the-ultimate-guide-to-angular-2025-coding-standards-clean-code-real-projects-15ad)

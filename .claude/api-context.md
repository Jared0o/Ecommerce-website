# API Context — Ecommerce Storefront

Backend: .NET 10 Modular Monolith  
Base URL: `http://localhost:5000/api`  
Docs (Scalar UI): `http://localhost:5000/scalar`

> Pełna dokumentacja API (z endpointami admina): `.claude/frontend/api-context.md`

---

## Autentykacja

### Flow

```
POST /auth/register               → rejestracja
POST /auth/login                  → { accessToken, refreshToken, expiresAt }
POST /auth/refresh                → { accessToken, refreshToken, expiresAt }  (body: { token })
POST /auth/revoke                 → wylogowanie (body: { token })
POST /auth/password/reset-request → wyślij email z linkiem resetu
POST /auth/password/reset         → zresetuj hasło tokenem
POST /auth/password/change        → zmień hasło (wymaga JWT)
```

### Token

- **Access token**: JWT Bearer, wygasa po **15 minut**
- **Refresh token**: wygasa po **7 dni**, przechowuj w `localStorage`
- Nagłówek: `Authorization: Bearer <accessToken>`

**TokenDto:**

```typescript
interface TokenDto {
  accessToken: string;
  refreshToken: string;
  expiresAt: string; // wygaśnięcie refresh tokenu (ISO 8601)
}
```

**Dekodowany payload JWT:**

```json
{
  "sub": "guid użytkownika",
  "email": "user@example.com",
  "role": ["User"],
  "exp": 1234567890
}
```

---

## Format błędów

```json
// Błąd domenowy
{ "type": "CartItemNotFoundError", "message": "..." }

// Błąd walidacji
{
  "type": "ValidationError",
  "message": "Validation failed.",
  "errors": [
    { "propertyName": "Quantity", "errorMessage": "Quantity must be between 1 and 999." }
  ]
}
```

**Kody HTTP:**

- `200` — OK
- `201` — Created (body = Guid)
- `204` — NoContent
- `400` — Bad Request
- `401` — Unauthorized
- `403` — Forbidden
- `404` — Not Found

---

## Paginacja

Parametry: `?page=1&pageSize=20`

```json
{
  "items": [...],
  "totalCount": 42,
  "page": 1,
  "pageSize": 20,
  "totalPages": 3
}
```

---

## Endpointy — Sklep

### Auth — `/auth`

| Method | Path                           | Auth | Opis                              |
| ------ | ------------------------------ | ---- | --------------------------------- |
| POST   | `/auth/register`               | ❌   | Rejestracja                       |
| POST   | `/auth/login`                  | ❌   | Logowanie → tokeny                |
| POST   | `/auth/refresh`                | ❌   | Odśwież token (body: `{ token }`) |
| POST   | `/auth/revoke`                 | ✅   | Wyloguj (body: `{ token }`)       |
| POST   | `/auth/password/reset-request` | ❌   | Wyślij link reset hasła           |
| POST   | `/auth/password/reset`         | ❌   | Zresetuj hasło tokenem            |
| POST   | `/auth/password/change`        | ✅   | Zmień hasło                       |

**POST `/auth/register`** — body:

```json
{
  "email": "string",
  "password": "string",
  "firstName": "string",
  "lastName": "string"
}
```

**POST `/auth/login`** — body:

```json
{ "email": "string", "password": "string" }
```

**POST `/auth/password/reset-request`** — body:

```json
{ "email": "string" }
```

**POST `/auth/password/reset`** — body:

```json
{ "token": "string", "newPassword": "string" }
```

**POST `/auth/password/change`** — body:

```json
{ "currentPassword": "string", "newPassword": "string" }
```

---

### Profil użytkownika — `/users`

| Method | Path                               | Auth | Opis                |
| ------ | ---------------------------------- | ---- | ------------------- |
| GET    | `/users/me`                        | ✅   | Pobierz profil      |
| PUT    | `/users/me`                        | ✅   | Zaktualizuj profil  |
| GET    | `/users/me/addresses`              | ✅   | Lista adresów       |
| POST   | `/users/me/addresses`              | ✅   | Dodaj adres         |
| PUT    | `/users/me/addresses/{id}`         | ✅   | Zaktualizuj adres   |
| DELETE | `/users/me/addresses/{id}`         | ✅   | Usuń adres          |
| PATCH  | `/users/me/addresses/{id}/default` | ✅   | Ustaw jako domyślny |

**PUT `/users/me`** — body:

```json
{
  "firstName": "string",
  "lastName": "string",
  "phoneNumber": "string | null",
  "avatarUrl": "string | null"
}
```

**POST/PUT `/users/me/addresses`** — body:

```json
{
  "label": "string",
  "street": "string",
  "buildingNumber": "string",
  "apartmentNumber": "string | null",
  "city": "string",
  "postalCode": "string",
  "country": "string"
}
```

---

### Koszyk — `/cart`

Wymagają JWT. Koszyk gościa trzymaj w `localStorage` i scalaj przez `POST /cart/merge` po zalogowaniu.

| Method | Path                   | Opis                                                    |
| ------ | ---------------------- | ------------------------------------------------------- |
| GET    | `/cart`                | Pobierz koszyk (zawsze zwraca, tworzy pusty jeśli brak) |
| POST   | `/cart/items`          | Dodaj produkt                                           |
| PUT    | `/cart/items/{itemId}` | Zmień ilość (0 = usuń)                                  |
| DELETE | `/cart/items/{itemId}` | Usuń item                                               |
| DELETE | `/cart`                | Wyczyść koszyk                                          |
| POST   | `/cart/merge`          | Scal koszyk gościa po zalogowaniu                       |

**POST `/cart/items`** — body:

```json
{
  "productId": "guid",
  "variantId": "guid",
  "quantity": 1,
  "productName": "string",
  "variantName": "string | null",
  "imageUrl": "string | null"
}
```

**PUT `/cart/items/{itemId}`** — body:

```json
{ "quantity": 2 }
```

**POST `/cart/merge`** — body:

```json
{
  "items": [{ "productId": "guid", "variantId": "guid", "quantity": 1 }]
}
```

---

### Katalog (publiczne) — `/catalog`

Brak autoryzacji. Tylko produkty ze statusem `Active`.

| Method | Path                         | Opis                |
| ------ | ---------------------------- | ------------------- |
| GET    | `/catalog/products`          | Lista produktów     |
| GET    | `/catalog/products/featured` | Wyróżnione produkty |
| GET    | `/catalog/products/{slug}`   | Szczegóły produktu  |
| GET    | `/catalog/categories`        | Drzewo kategorii    |
| GET    | `/catalog/categories/{slug}` | Kategoria po slug   |
| GET    | `/catalog/brands`            | Lista marek         |
| GET    | `/catalog/brands/{slug}`     | Marka po slug       |

**GET `/catalog/products`** — query params:

```
?page=1&pageSize=20&categoryId={guid}&brandId={guid}&minPrice={decimal}&maxPrice={decimal}
```

**GET `/catalog/brands`** — query params:

```
?page=1&pageSize=20
```

---

## Typy

### ProductListItemDto

```typescript
interface ProductListItemDto {
  id: string;
  name: string;
  slug: string;
  mainCategoryId: string;
  brandId: string | null;
  price: number | null; // ZAWSZE null — cena jest na wariantach, użyj GET /{slug}
  isFeatured: boolean;
  tags: string[];
  createdAt: string;
}
```

### ProductDto (szczegóły — GET /catalog/products/{slug})

```typescript
interface ProductDto {
  id: string;
  name: string;
  slug: string;
  description: string | null;
  shortDescription: string | null;
  mainCategoryId: string;
  brandId: string | null;
  isFeatured: boolean;
  tags: string[];
  createdAt: string;
  updatedAt: string;
  variants: ProductVariantDto[];
  images: ProductImageDto[];
  attributes: ProductAttributeValueDto[];
}

interface ProductVariantDto {
  id: string;
  sku: string;
  name: string | null;
  price: number; // cena tutaj — nie na ProductDto
  compareAtPrice: number | null;
  isDefault: boolean;
  attributes: { key: string; value: string }[];
}

interface ProductImageDto {
  id: string;
  variantId: string | null; // null = globalne zdjęcie produktu
  url: string;
  altText: string | null;
  sortOrder: number;
}

interface ProductAttributeValueDto {
  attributeDefinitionId: string;
  name: string; // np. "Materiał"
  value: string; // np. "Bawełna"
}
```

### CategoryDto (drzewo)

```typescript
interface CategoryDto {
  id: string;
  name: string;
  slug: string;
  parentId: string | null;
  level: 1 | 2 | 3;
  description: string | null;
  imageUrl: string | null;
  sortOrder: number;
  isActive: boolean;
  createdAt: string;
  attributes: AttributeDefinitionDto[];
  children: CategoryDto[];
}
```

### BrandDto

```typescript
interface BrandDto {
  id: string;
  name: string;
  slug: string;
  logoUrl: string | null;
  isActive: boolean;
  createdAt: string;
}
```

### CartDto

```typescript
interface CartDto {
  userId: string;
  expiresAt: string; // koszyk wygasa 7 dni od ostatniej modyfikacji
  items: CartItemDto[];
  totalValue: number;
}

interface CartItemDto {
  id: string;
  productId: string;
  variantId: string;
  productName: string;
  variantName: string | null;
  imageUrl: string | null;
  quantity: number;
  currentPrice: number; // live z katalogu — nie cachować
  totalPrice: number; // currentPrice * quantity
  addedAt: string;
}
```

---

## Reguły biznesowe dla UI

### Ceny

| Reguła                                                          | Implementacja                                                                            |
| --------------------------------------------------------------- | ---------------------------------------------------------------------------------------- |
| `ProductListItemDto.price` jest zawsze `null`                   | Pokaż cenę z `variants.find(v => v.isDefault)?.price` — tylko na stronie produktu        |
| Na liście produktów cena jest niedostępna bez drugiego requesta | Opcje: brak ceny na karcie; lub fetch szczegółów dla widocznych kart (niepolecane — N+1) |
| `compareAtPrice` > `price` → promocja                           | Pokaż przekreśloną `compareAtPrice`                                                      |

### Koszyk

| Reguła                                                 | Implementacja                                                                                          |
| ------------------------------------------------------ | ------------------------------------------------------------------------------------------------------ |
| Koszyk gościa w `localStorage`                         | Klucz: `guestCart`, format: `{ productId, variantId, quantity, productName, variantName, imageUrl }[]` |
| Scal po zalogowaniu                                    | `POST /cart/merge` → `invalidateQueries(['cart'])` → `clearGuestCart()`                                |
| `GET /cart` zawsze bezpieczny                          | Nigdy nie zwraca 404 — tworzy pusty koszyk automatycznie                                               |
| Nie sumuj itemów lokalnie                              | Po `POST /cart/items` zawsze odśwież przez re-fetch — backend sumuje duplikaty                         |
| Koszyk wygasa                                          | Jeśli `expiresAt < now + 24h` → pokaż banner "Koszyk wygaśnie wkrótce"                                 |
| Po złożeniu zamówienia koszyk czyszczony automatycznie | Wymuś `invalidateQueries(['cart'])` po sukcesie zamówienia                                             |
| Max 999 sztuk jednego wariantu                         | Blokuj input na poziomie UI; obsługuj `InvalidQuantityError`                                           |

### Nawigacja i SEO

| Reguła                                               | Implementacja                                                          |
| ---------------------------------------------------- | ---------------------------------------------------------------------- |
| Slug jako canonical URL                              | Zawsze nawiguj przez `slug`, nie `id`                                  |
| Generuj `metadata` na stronach produktów i kategorii | `generateMetadata` w każdym `page.tsx`                                 |
| Kategorie są hierarchiczne (max 3 poziomy)           | Breadcrumbs z `parentId` chain                                         |
| `isActive: false` na kategorii/marce                 | Backend nie zwraca ich w publicznych endpointach — nie filtruj ręcznie |

---

## Błędy domenowe

| Type                      | HTTP | Kiedy                                                       |
| ------------------------- | ---- | ----------------------------------------------------------- |
| `InvalidCredentialsError` | 401  | Błędny email lub hasło                                      |
| `CartItemNotFoundError`   | 404  | Item nie istnieje lub należy do innego użytkownika          |
| `InvalidQuantityError`    | 400  | Ilość ≤ 0 lub > 999                                         |
| `ProductNotFoundError`    | 404  | Produkt nie istnieje (obsługuj jako `notFound()` w Next.js) |
| `CategoryNotFoundError`   | 404  | Kategoria nie istnieje                                      |
| `BrandNotFoundError`      | 404  | Marka nie istnieje                                          |
| `ValidationError`         | 400  | Błędy walidacji pól (zawiera `errors[]`)                    |

---

## CORS

Backend podczas developmentu akceptuje wszystkie originy.
